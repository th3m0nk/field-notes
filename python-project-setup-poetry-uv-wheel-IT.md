# Gestione dei progetti Python moderni: Poetry, uv e le wheel

Negli ultimi anni l’ecosistema Python ha fatto un grande passo avanti nella gestione delle dipendenze, degli ambienti virtuali e del packaging. Strumenti come **Poetry** e **uv** nascono per risolvere problemi storici di riproducibilità, performance e complessità; mentre il formato **wheel** è centrale per un’installazione affidabile e veloce dei pacchetti. 

> **Nota**: per ogni concetto citato in questo articolo trovi i riferimenti alle **fonti ufficiali** (documentazione e repository GitHub) nelle sezioni dedicate e nella sezione finale “Riferimenti ufficiali”.

---

## 1) `pyproject.toml`: il punto di partenza

Sia Poetry che uv ruotano attorno a **`pyproject.toml`**, il file di configurazione standard per definire metadati di progetto, dipendenze e build backend.

In pratica, `pyproject.toml` può contenere:

- metadati del progetto (nome, versione, descrizione)
- dipendenze e relativi vincoli
- configurazione del sistema di build
- configurazioni dei tool (formatter, linter, test, ecc.)

---

## 2) Cos’è Poetry

**Poetry** è un **tool per dependency management e packaging**. Consente di dichiarare le dipendenze del progetto, installarle/aggiornarle in modo deterministico grazie a un lockfile, e costruire artefatti di distribuzione (sdist e wheel). 

### Funzionalità principali

- **Gestione dipendenze**: aggiunta/rimozione/aggiornamento con risoluzione dei vincoli.
- **Lockfile**: genera **`poetry.lock`** per installazioni riproducibili.
- **Ambienti isolati**: può creare/gestire virtualenv per progetto.
- **Packaging**: costruzione di **sdist** e **wheel** e supporto al publish.

### File chiave

- `pyproject.toml`: sorgente di verità (metadati e dipendenze) nel formato gestito da Poetry.
- `poetry.lock`: snapshot delle versioni risolte e installate.

### Esempio minimo

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

### Risorse ufficiali Poetry

- Documentazione: https://python-poetry.org/docs/
- Sito ufficiale: https://python-poetry.org/
- GitHub (repository principale): https://github.com/python-poetry/poetry
- Pagina PyPI: https://pypi.org/project/poetry/

---

## 3) Cos’è uv

**uv** è un **package e project manager estremamente veloce**, scritto in **Rust**, focalizzato su:

- risoluzione dipendenze
- installazione pacchetti
- gestione ambienti virtuali
- lockfile universale

È progettato per essere molto rapido e per offrire una CLI moderna e pragmatica.

### Cosa fa uv (in pratica)

- crea virtualenv
- aggiunge e sincronizza dipendenze
- genera lockfile
- offre anche una modalità “pip-compatibile” (utile per migrazioni o abitudini consolidate)

Esempio tipico:

```bash
uv init example
cd example
uv add ruff
uv lock
uv sync
uv run ruff check
```

### Risorse ufficiali uv

- Documentazione: https://docs.astral.sh/uv/
- Installazione: https://docs.astral.sh/uv/getting-started/installation/
- GitHub (repository principale): https://github.com/astral-sh/uv
- GitHub Action ufficiale: https://github.com/astral-sh/setup-uv
- Guida GitHub Actions: https://docs.astral.sh/uv/guides/integration/github/

---

## 4) Cos’è una wheel

Una **wheel** è un formato di distribuzione binaria per Python (`.whl`), definito dallo standard Wheel (PEP 427). Una wheel è un archivio ZIP con naming convenzionale, pensato per essere **installato direttamente** senza dover compilare o eseguire logica di build durante l’installazione.

### Perché le wheel sono importanti

- **Installazioni più veloci**: l’installer copia i file in posizione, evitando build ripetute.
- **Meno errori in CI e su macchine “minimali”**: spesso non serve un compilatore.
- **Maggiore riproducibilità**: stesso artefatto, stesso risultato (a parità di compatibilità).

### Come leggere il nome di una wheel

Esempio:

```text
numpy-1.26.4-cp311-cp311-manylinux_2_17_x86_64.whl
```

La convenzione include:
- nome e versione
- tag Python (es. `cp311`)
- tag ABI
- tag piattaforma (es. `manylinux`, `win_amd64`, ecc.)

### Wheel “universali”

```text
requests-2.32.0-py3-none-any.whl
```

Indica pacchetto “pure Python” installabile su qualsiasi OS per Python 3.

### Risorse ufficiali wheel

- Specifica storica: PEP 427 (Wheel Binary Package Format): https://peps.python.org/pep-0427/
- Tool di riferimento “wheel”: docs: https://wheel.readthedocs.io/en/stable/
- Pagina PyPI del pacchetto `wheel`: https://pypi.org/project/wheel/
- Documentazione pip (`pip wheel`): https://pip.pypa.io/en/stable/cli/pip_wheel.html

---

## 5) Come Poetry e uv usano le wheel

Sia Poetry che uv, quando installano dipendenze:

- **preferiscono una wheel compatibile**, se disponibile
- usano la source distribution solo se non esistono wheel adatte

Questo vale in particolare per pacchetti con estensioni native (es. numpy/pandas), dove la disponibilità di wheel evita compilazioni e riduce drasticamente i problemi in ambienti CI/CD.

---

## 6) Confronto diretto: Poetry vs uv

### Filosofia

| Aspetto | Poetry | uv |
|---|---|---|
| Approccio | “Project manager” completo (dependency + packaging + publish) | Tool veloce e modulare per dipendenze/ambienti |
| Packaging & publish | Integrati | Supportati, ma spesso in pipeline con tool dedicati |
| “Opinionated” | Sì (workflow e convenzioni) | Meno (più aderente a standard e componibile) |

### Performance

In generale uv punta a essere molto più veloce nell’installazione e nella risoluzione rispetto agli approcci tradizionali.

### Lockfile

- Poetry: `poetry.lock`
- uv: `uv.lock`

Entrambi abilitano installazioni riproducibili.

---

## 7) Quando scegliere Poetry

Scegli **Poetry** se:

- stai costruendo una **libreria** da distribuire
- vuoi un tool unico “end-to-end” (dipendenze + build + publish)
- preferisci un workflow guidato e coerente

---

## 8) Quando scegliere uv

Scegli **uv** se:

- sviluppi **applicazioni**, ETL, microservizi, tool interni
- dai priorità a velocità in **CI/CD** e ambienti puliti
- vuoi un tool “drop-in” (anche pip-compatibile) e componibile

---

## 9) Conclusione

- **Poetry**: project manager completo per dipendenze e packaging.
- **uv**: tool modernissimo e velocissimo per dipendenze, ambienti e lockfile.
- **Wheel**: formato fondamentale che abilita installazioni rapide e affidabili.

Molti team adottano oggi un approccio ibrido:

> **uv per dipendenze/ambienti + tool specifici per build/publish**, quando serve.

---

## Riferimenti ufficiali (link rapidi)

### Poetry
- Sito: https://python-poetry.org/
- Docs: https://python-poetry.org/docs/
- GitHub: https://github.com/python-poetry/poetry
- PyPI: https://pypi.org/project/poetry/

### uv (Astral)
- Docs: https://docs.astral.sh/uv/
- Installazione: https://docs.astral.sh/uv/getting-started/installation/
- GitHub: https://github.com/astral-sh/uv
- setup-uv (GitHub Action): https://github.com/astral-sh/setup-uv
- Guida GitHub Actions: https://docs.astral.sh/uv/guides/integration/github/

### Wheel
- PEP 427: https://peps.python.org/pep-0427/
- Docs `wheel`: https://wheel.readthedocs.io/en/stable/
- PyPI `wheel`: https://pypi.org/project/wheel/
- pip `wheel`: https://pip.pypa.io/en/stable/cli/pip_wheel.html
