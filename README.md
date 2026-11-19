# OpenSecGate

**Design and Evaluation of an Open-Source Security Gate for CI/CD Pipelines Using Context-Aware Vulnerability Prioritisation**

OpenSecGate is a planned Python 3.11 command-line tool and GitHub Action. It brings together findings from existing security scanners, adds project context and decides whether a CI/CD pipeline should continue or stop.

Most CI/CD security checks depend heavily on the severity reported by a scanner. This can be a problem because a high severity score does not always mean that a vulnerability can affect the project. At the same time, a lower-rated finding may still be dangerous when it involves an exposed secret or an unsafe workflow. OpenSecGate is designed to make these decisions more useful and easier to explain.

The aim is **not to create another vulnerability scanner**. OpenSecGate will use four open-source tools, normalise their findings and apply a policy that considers both severity and the context in which the problem was found.

> **Project status:** The Python project structure and development-tool configuration are in place. The common finding schema, scanner adapters, decision engine, command-line interface and GitHub Action are not yet implemented.

## Development logbook

Development progress, decisions, testing evidence and next steps are recorded in the [OpenSecGate Development Logbook](docs/LOGBOOK.md). Each entry is dated and distinguishes completed work from planned work.

## How OpenSecGate will work

The design has two main layers:

1. **Evidence layer.** Existing scanners check source code, dependencies, exposed secrets and GitHub Actions workflows.
2. **Decision layer.** OpenSecGate converts their output into one finding format, adds six context signals and uses the weights and thresholds in `policy.yml` to decide what should happen to the build.

```text
Push or pull request
        |
        v
GitHub Actions workflow
        |
        +---- Semgrep --------- Insecure code patterns
        +---- OSV-Scanner ----- Vulnerable dependencies
        +---- Gitleaks -------- Exposed secrets
        +---- zizmor ---------- Workflow misconfigurations
        |
        v
Normalise findings into one schema
        |
        v
Add six context signals
        |
        v
Evaluate using policy.yml
        |
        +---- PASS  -> Continue
        +---- WARN  -> Continue with a warning
        +---- BLOCK -> Stop the pipeline
        |
        v
Written reasons and reports
```

The planned outputs are a written reason for each decision, a pull-request summary, an HTML report and SARIF findings for GitHub's Security tab. These are development targets, not features available in the current repository.

## Six context signals

OpenSecGate will consider the following signals where they are relevant to a finding.

| Context signal | What it checks |
| --- | --- |
| Direct or transitive | Is a dependency declared by the project, or brought in by another package? |
| Production or development | Is the dependency used in the released application or only for development and testing? |
| Imported in code | Does the source code import the affected package? |
| Known exploitation | Is the vulnerability in CISA's Known Exploited Vulnerabilities (KEV) catalogue, and what does EPSS indicate? |
| Secret location | Is a detected secret in live code or in a test fixture? |
| Workflow privilege | Does a workflow use write permissions, risky triggers or unpinned actions? |

These signals provide practical approximations to some of the questions raised by full reachability and exploitability analysis. They are not a replacement for full reachability analysis.

### Decisions and safety rule

The policy file will contain the weights and thresholds used to return **PASS**, **WARN** or **BLOCK**, along with an explanation.

A fixed safety rule will take priority over the weighted score: **a KEV-listed vulnerability or a live secret must result in BLOCK**. A failed scanner, unreadable report or failed policy evaluation must not be treated as a clean PASS.

## Project scope

The initial version will support:

- JavaScript projects using npm and Python projects using PyPI.
- GitHub Actions as the CI/CD platform.
- Findings from Semgrep, OSV-Scanner, Gitleaks and zizmor.
- Rule-based decisions using a configuration file.

The project will not develop new vulnerability detection methods, provide a web interface, use machine learning or large language models, perform full reachability analysis, support other CI platforms or automatically fix vulnerabilities.

## Development requirements

### Tools needed for the first stage

| Tool | Purpose |
| --- | --- |
| Python 3.11 | Main programming language and runtime |
| Git | Version control |
| `venv` and `pip` | Python environment and package management |
| `pytest` | Tests for adapters, validation and decision logic |
| `ruff` | Python linting and formatting |

Python's built-in `json`, `pathlib`, `dataclasses` and `typing` modules are sufficient for the first SARIF-reading stage. **No third-party runtime dependency is required just to read a SARIF JSON file.**

Further dependencies will be added when needed. For example, `PyYAML` is a candidate for reading `policy.yml` safely, and a packaging backend will be declared when `pyproject.toml` is created. Semgrep, OSV-Scanner, Gitleaks and zizmor will be installed or invoked separately as their integrations are developed.

### Local setup

Python 3.11 and Git must be installed first. The following commands create a development environment; they do not install a finished OpenSecGate application.

**Linux and macOS**

```bash
git clone https://github.com/ymmiah/OpenSecGate.git
cd OpenSecGate
python3.11 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -e ".[dev]"
python --version
```

**Windows PowerShell**

```powershell
git clone https://github.com/ymmiah/OpenSecGate.git
cd OpenSecGate
py -3.11 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -e ".[dev]"
python --version
```

The Python version should be **3.11.x**. The virtual environment (`.venv/`) is excluded from version control by `.gitignore`.

## First development stage: Semgrep SARIF adapter

The first task is to read a saved **Semgrep SARIF 2.1.0** report and convert its results into OpenSecGate's common finding format. Starting with stored scanner output will make it possible to test the adapter without running a new scan for every test.

SARIF is a JSON-based format. The fields needed at this stage include `version`, `runs`, `results`, `ruleId`, `message`, `level` and any available file locations or rule metadata. Some details are optional, so missing locations and other optional fields must be handled without inventing information.

The common finding format is planned to include the scanner name, rule identifier, message, severity, file path, line number, stable identifier and relevant scanner metadata. Its final field types will be defined alongside the first tests.

### Initial test cases

The adapter will be checked against:

- A valid SARIF report with one finding.
- A valid report with no findings.
- Several findings across more than one run.
- Missing optional fields, such as a file location or line number.
- Invalid JSON, unsupported SARIF versions and malformed required data.
- Consistent mapping of scanner levels and rule metadata.

A valid report with no findings is different from a report that failed to load. The latter must return an error and must not be presented as a successful scan.

After the source and test files have been created, the planned local checks are:

```bash
python -m pytest -q
python -m ruff check .
```

No test results are claimed at this stage, as the implementation and test suite are still to be written.

### Planned project layout

```text
OpenSecGate/
├── README.md
├── .gitignore
├── pyproject.toml
├── policy.yml
├── src/
│   └── opensecgate/
│       ├── __init__.py
│       ├── models.py
│       └── adapters/
│           ├── __init__.py
│           ├── semgrep.py
│           ├── osv.py
│           ├── gitleaks.py
│           └── zizmor.py
├── tests/
│   ├── fixtures/
│   └── test_semgrep_adapter.py
└── action.yml
```

The project configuration, source package and test directory have been created. The finding model, adapter modules, policy and GitHub Action will be added in later tasks.

## Development roadmap

| Stage | Work | Expected result |
| --- | --- | --- |
| 1. Setup and common format | Configure Python 3.11, project packaging and tests; define the normalised finding schema | A consistent basis for the four adapters |
| 2. Scanner integration | Start with Semgrep SARIF, then add OSV-Scanner, Gitleaks and zizmor | One list of findings from all four scanners |
| 3. Context enrichment | Add the six context signals | Findings contain relevant project context |
| 4. Decision engine | Read `policy.yml`, apply weights, thresholds and the fixed safety rule | Explainable PASS, WARN or BLOCK decisions |
| 5. GitHub Action | Add the CLI entry point, workflow integration and reporting | Version 1.0 usable in GitHub Actions |
| 6. Evaluation | Compare decisions with a severity-only gate | Evidence of strengths, limitations and pipeline overhead |

The interim project schedule sets **12 February 2027** as the planned code-freeze date, with evaluation to follow. The timetable may be revised if development or testing shows that a task needs more work.

## Evaluation plan

OpenSecGate will be compared with a severity-only gate using **60 labelled test cases** across the four scanners. These will include **10 context traps** where context might appear to lower the risk even though the correct decision remains BLOCK.

Both gates will be tested on the same stored scanner findings, so the decision logic is the main difference. The evaluation will measure:

- Precision, recall and F1-score.
- The rate at which high-risk cases are missed.
- How decisions change when signal weights are adjusted by ±50%.
- What happens when one context signal is removed at a time.
- Additional pipeline time, measured across 5 to 10 public repositories.

The benchmark labels will be established before the gates are evaluated. A sample will also be independently labelled so agreement can be measured using Cohen's kappa.

The results may show that context-aware decisions improve prioritisation, but that is something the project must **test rather than assume**.

## Security and handling of findings

The project will use publicly available code and saved scanner reports for testing. It will not probe systems without permission. Test fixtures must not contain real secrets or unnecessary personal data.

Scanner tools and their output formats may change. Their versions and test reports will therefore be recorded so results can be reproduced. GitHub Actions permissions will be kept to the minimum needed, with any SARIF upload permission granted only where required.

## Technical references

- [SARIF 2.1.0 specification — OASIS](https://docs.oasis-open.org/sarif/sarif/v2.1.0/sarif-v2.1.0.html)
- [Semgrep — local and CLI scans](https://semgrep.dev/docs/category/local-and-cli-scans)
- [GitHub Actions documentation](https://docs.github.com/en/actions)
- [Uploading SARIF files to GitHub code scanning](https://docs.github.com/en/code-security/how-tos/find-and-fix-code-vulnerabilities/integrate-with-existing-tools/upload-sarif-file)
- [Python Packaging User Guide](https://packaging.python.org/en/latest/guides/writing-pyproject-toml/)
- [pytest documentation](https://docs.pytest.org/)
- [Ruff documentation](https://docs.astral.sh/ruff/)

---

**Current status:** Stage 1 — Python project structure prepared. The common finding schema and its tests are next, followed by the Semgrep SARIF adapter in Stage 2.
