# Poetry, uv, and wheels: what you need to know to manage a Python project

Dependency management in Python has been an open problem for years. The classic workflow — `venv` + `pip` + `requirements.txt` — works, but doesn't scale: it doesn't handle version conflicts, doesn't guarantee reproducibility across machines, and leaves the developer responsible for manually keeping the environment in sync with what's declared in the requirements file.

Poetry and uv exist to solve this problem. Wheels are the distribution format that makes package installation fast and predictable. This article explains how they work and when to use one over the other.

---

## `pyproject.toml`: the central configuration file

Everything revolves around **`pyproject.toml`**, the standard defined by [PEP 621](https://peps.python.org/pep-0621/) for Python project metadata. It contains: project name, version, dependencies, and tool configuration (formatter, linter, test runner).

Both Poetry and uv read and write this file. If you're coming from `requirements.txt`, expect a paradigm shift: dependencies are no longer a flat list but a structured declaration with version constraints that the tool resolves automatically.

---

## Poetry

Poetry handles dependencies, virtual environments, and packaging in a single tool. Each project gets an isolated environment and a **`poetry.lock`** file that pins exact versions of every package. By sharing the lockfile, anyone can reconstruct the exact same environment.

A `pyproject.toml` managed by Poetry:

```toml
[tool.poetry]
name = "my-project"
version = "0.1.0"
description = "Example"

[tool.poetry.dependencies]
python = "^3.11"
requests = "^2.32.0"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

Poetry's main advantage is the full cycle: dependency resolution → virtual environment → artifact build (sdist and wheel) → publish to PyPI. For those developing **libraries** for distribution, having everything integrated reduces configuration overhead and potential breakage points.

The downsides: the resolver is slower than uv's, the CLI has more concepts to learn, and virtual environment management can be confusing regarding where environments physically reside (by default outside the project directory, changeable with `virtualenvs.in-project = true`).

**Docs:** [python-poetry.org/docs](https://python-poetry.org/docs/) · [GitHub](https://github.com/python-poetry/poetry)

---

## uv

uv is a tool by Astral (the same team behind Ruff), written in Rust. It handles dependency resolution and environment management like Poetry, but with drastically shorter install times — on the order of tens of times faster on projects with many dependencies.

Typical workflow:

```bash
uv init example
cd example
uv add ruff
uv lock
uv sync
uv run ruff check
```

It generates a lockfile (`uv.lock`) and offers a pip-compatible mode (`uv pip install`), which eases migration from existing workflows.

Where uv falls short: it has no built-in support for package build and publish. If you need to publish a library to PyPI, you'll need an external tool (e.g. `build` + `twine`, or `flit`). For applications, services, scripts, and CI/CD pipelines this isn't an issue — and uv's speed becomes a tangible advantage.

**Docs:** [docs.astral.sh/uv](https://docs.astral.sh/uv/) · [GitHub](https://github.com/astral-sh/uv)

---

## Wheels: the distribution format

When a package manager installs a package, it obtains it in one of two forms:

- **sdist (source distribution)**: source code to be compiled at install time. Requires compilers and, for packages with C/Rust extensions, the relevant toolchains.
- **wheel (`.whl`)**: precompiled archive, ready to be extracted. No compilation, deterministic result.

A wheel filename encodes compatibility:

```text
numpy-1.26.4-cp311-cp311-manylinux_2_17_x86_64.whl
```

This means: numpy 1.26.4, CPython 3.11, Linux x86_64. If the platform matches, installation is a file copy. If no compatible wheel exists, the package manager falls back to the sdist — and that's where things can break (missing compilers, absent system headers, cryptic errors).

Pure Python packages use universal wheels:

```text
requests-2.32.0-py3-none-any.whl
```

`py3-none-any`: any Python 3 implementation, no ABI requirement, any platform.

Both Poetry and uv prefer wheels when available. This isn't something you need to manage manually — but it's useful to know that when an installation fails with compilation errors, the reason is almost always the absence of a wheel for your platform.

**Spec:** [PEP 427](https://peps.python.org/pep-0427/) · [Docs](https://wheel.readthedocs.io/en/stable/)

---

## Which one to choose

The choice depends on what you're building.

**Library to publish on PyPI** → Poetry. The integrated build-publish cycle saves you from assembling a toolchain by hand.

**Application, service, script, CI/CD pipeline** → uv. Faster, less ceremony, gradual migration from pip possible.

| | Poetry | uv |
|---|---|---|
| Built-in build and publish | Yes | No |
| Resolver speed | Adequate | Very fast |
| Learning curve | More concepts to absorb | More immediate |
| Lockfile | `poetry.lock` | `uv.lock` |
| pip compatibility | No | Yes (`uv pip`) |

A note on migration: both tools use `pyproject.toml`, so switching from one to the other doesn't require rewriting the project. The lockfile needs to be regenerated, but the declared dependencies stay the same.

---

## Common issues for beginners

- **"Python not found" after installation.** On Windows, the Python installed from the Microsoft Store and the one from python.org have different paths. Check with `python --version` which one responds, and whether it's the one you expect.
- **Virtual environment conflicts.** If you used `pip install` globally before adopting Poetry or uv, global packages can interfere. Always work in an isolated virtual environment.
- **Compilation errors during installation.** This means no wheel exists for the package on your platform. The options are: find a version of the package that has wheels available, or install the necessary compilers (Visual C++ Build Tools on Windows, `build-essential` on Debian/Ubuntu).
- **Lockfile not committed to the repository.** The lockfile (`poetry.lock` or `uv.lock`) should be versioned. Without it, each `install` can produce a different environment.

---

## References

### Poetry
- Docs: https://python-poetry.org/docs/
- GitHub: https://github.com/python-poetry/poetry

### uv
- Docs: https://docs.astral.sh/uv/
- Installation: https://docs.astral.sh/uv/getting-started/installation/
- GitHub: https://github.com/astral-sh/uv

### Wheel
- PEP 427: https://peps.python.org/pep-0427/
- Docs: https://wheel.readthedocs.io/en/stable/
