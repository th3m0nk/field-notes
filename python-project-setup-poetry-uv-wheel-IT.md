# Poetry, uv e wheel: cosa serve sapere per gestire un progetto Python

La gestione delle dipendenze in Python è stata un problema aperto per anni. Il workflow classico — `venv` + `pip` + `requirements.txt` — funziona, ma non scala: non gestisce conflitti tra versioni, non garantisce riproducibilità tra macchine diverse, e lascia al developer il compito di tenere allineato manualmente l'ambiente con ciò che è dichiarato nel file di requisiti.

Poetry e uv esistono per risolvere questo problema. Le wheel sono il formato di distribuzione che rende l'installazione dei pacchetti veloce e prevedibile. Questo articolo spiega come funzionano e quando usare l'uno o l'altro.

---

## `pyproject.toml`: il file di configurazione centrale

Tutto ruota attorno a **`pyproject.toml`**, lo standard definito da [PEP 621](https://peps.python.org/pep-0621/) per i metadati dei progetti Python. Contiene: nome del progetto, versione, dipendenze, e configurazione degli strumenti (formatter, linter, test runner).

Sia Poetry che uv leggono e scrivono questo file. Chi arriva da `requirements.txt` deve abituarsi a un cambio di paradigma: le dipendenze non sono più una lista piatta, ma una dichiarazione strutturata con vincoli di versione che il tool risolve automaticamente.

---

## Poetry

Poetry gestisce dipendenze, ambienti virtuali e packaging in un unico tool. Ogni progetto ottiene un ambiente isolato e un file **`poetry.lock`** che fissa le versioni esatte di ogni pacchetto. Condividendo il lockfile, chiunque ricostruisce lo stesso identico ambiente.

Un `pyproject.toml` gestito da Poetry:

```toml
[tool.poetry]
name = "mio-progetto"
version = "0.1.0"
description = "Esempio"

[tool.poetry.dependencies]
python = "^3.11"
requests = "^2.32.0"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

Il vantaggio principale di Poetry è il ciclo completo: risoluzione dipendenze → ambiente virtuale → build degli artefatti (sdist e wheel) → pubblicazione su PyPI. Per chi sviluppa **librerie** da distribuire, avere tutto integrato riduce la quantità di configurazione e i punti di rottura.

I limiti: il resolver è più lento rispetto a uv, la CLI ha più concetti da imparare, e la gestione degli ambienti virtuali può creare confusione su dove risiedono fisicamente (di default fuori dalla directory del progetto, modificabile con `virtualenvs.in-project = true`).

**Docs:** [python-poetry.org/docs](https://python-poetry.org/docs/) · [GitHub](https://github.com/python-poetry/poetry)

---

## uv

uv è un tool di Astral (gli stessi di Ruff), scritto in Rust. Fa risoluzione delle dipendenze e gestione degli ambienti come Poetry, ma con tempi di installazione drasticamente inferiori — nell'ordine delle decine di volte più veloce su progetti con molte dipendenze.

Flusso di lavoro tipico:

```bash
uv init example
cd example
uv add ruff
uv lock
uv sync
uv run ruff check
```

Genera un lockfile (`uv.lock`) e offre una modalità compatibile con pip (`uv pip install`), che facilita la migrazione da workflow esistenti.

Dove uv è più debole: non ha un supporto integrato per build e publish di pacchetti. Se devi pubblicare una libreria su PyPI, serve un tool esterno (ad esempio `build` + `twine`, oppure `flit`). Per applicazioni, servizi, script e pipeline CI/CD questo non è un problema — e la velocità di uv diventa un vantaggio concreto.

**Docs:** [docs.astral.sh/uv](https://docs.astral.sh/uv/) · [GitHub](https://github.com/astral-sh/uv)

---

## Wheel: il formato di distribuzione

Quando un package manager installa un pacchetto, lo ottiene in una di due forme:

- **sdist (source distribution)**: codice sorgente da compilare al momento dell'installazione. Richiede compilatori e, per pacchetti con estensioni C/Rust, le relative toolchain.
- **wheel (`.whl`)**: archivio precompilato, pronto per essere estratto. Nessuna compilazione, risultato deterministico.

Il nome di una wheel codifica la compatibilità:

```text
numpy-1.26.4-cp311-cp311-manylinux_2_17_x86_64.whl
```

Questo significa: numpy 1.26.4, CPython 3.11, Linux x86_64. Se la piattaforma corrisponde, l'installazione è una copia di file. Se non esiste una wheel compatibile, il package manager cade sulla sdist — e lì le cose possono rompersi (compilatori mancanti, header di sistema assenti, errori criptici).

I pacchetti pure Python usano wheel universali:

```text
requests-2.32.0-py3-none-any.whl
```

`py3-none-any`: qualsiasi implementazione Python 3, nessun requisito ABI, qualsiasi piattaforma.

Sia Poetry che uv preferiscono le wheel quando disponibili. Non è qualcosa su cui intervenire manualmente — ma è utile sapere che, quando un'installazione fallisce con errori di compilazione, il motivo è quasi sempre l'assenza di una wheel per la propria piattaforma.

**Spec:** [PEP 427](https://peps.python.org/pep-0427/) · [Docs](https://wheel.readthedocs.io/en/stable/)

---

## Quale scegliere

La scelta dipende da cosa si sta costruendo.

**Libreria da pubblicare su PyPI** → Poetry. Il ciclo build-publish integrato evita di assemblare una toolchain a mano.

**Applicazione, servizio, script, pipeline CI/CD** → uv. Più veloce, meno cerimonia, migrazione graduale da pip possibile.

| | Poetry | uv |
|---|---|---|
| Build e publish integrati | Sì | No |
| Velocità del resolver | Adeguata | Molto alta |
| Curva di apprendimento | Più concetti da assimilare | Più immediato |
| Lockfile | `poetry.lock` | `uv.lock` |
| Compatibilità pip | No | Sì (`uv pip`) |

Una nota sulla migrazione: entrambi i tool usano `pyproject.toml`, quindi passare dall'uno all'altro non richiede riscrivere il progetto. Il lockfile va rigenerato, ma le dipendenze dichiarate restano le stesse.

---

## Problemi frequenti per chi inizia

- **"Python non trovato" dopo l'installazione.** Su Windows, il Python installato dal Microsoft Store e quello installato da python.org hanno path diversi. Verificare con `python --version` quale risponde, e che sia quello atteso.
- **Conflitti tra ambienti virtuali.** Se si è usato `pip install` globalmente prima di adottare Poetry o uv, i pacchetti globali possono interferire. Lavorare sempre in un ambiente virtuale isolato.
- **Errori di compilazione all'installazione.** Significa che non esiste una wheel per il pacchetto su quella piattaforma. Le opzioni sono: cercare una versione del pacchetto che abbia wheel disponibili, oppure installare i compilatori necessari (Visual C++ Build Tools su Windows, `build-essential` su Debian/Ubuntu).
- **Lockfile non committato nel repository.** Il lockfile (`poetry.lock` o `uv.lock`) va versionato. Senza di esso, ogni `install` può produrre un ambiente diverso.

---

## Riferimenti

### Poetry
- Docs: https://python-poetry.org/docs/
- GitHub: https://github.com/python-poetry/poetry

### uv
- Docs: https://docs.astral.sh/uv/
- Installazione: https://docs.astral.sh/uv/getting-started/installation/
- GitHub: https://github.com/astral-sh/uv

### Wheel
- PEP 427: https://peps.python.org/pep-0427/
- Docs: https://wheel.readthedocs.io/en/stable/
