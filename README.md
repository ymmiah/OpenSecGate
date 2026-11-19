# OpenSecGate

**Design and Evaluation of an Open-Source Security Gate for CI/CD Pipelines Using Context-Aware Vulnerability Prioritisation**

> **Status:** Final-year BSc Computing project — planning and initial development. This repository currently contains documentation only. The CLI, adapters, tests, policy engine and GitHub Action have **not yet been implemented**.

## Project aim

OpenSecGate is a planned **Python 3.11 command-line tool and GitHub Action** that will combine security findings from **Semgrep**, **OSV-Scanner**, **Gitleaks** and **zizmor**, normalise them into a common finding schema, assess six contextual risk signals, and apply configurable weights from `policy.yml` to produce a **PASS**, **WARN** or **BLOCK** decision.

The intended processing flow is:

```text
Scanner output files
    |
    +-- Semgrep (SARIF)           <-- FIRST DEVELOPMENT MILESTONE
    +-- OSV-Scanner
    +-- Gitleaks
    +-- zizmor
    |
    v
Format-specific adapters
    |
    v
Common finding schema
    |
    v
Six context signals + weighted policy.yml
    |
    v
Risk evaluation --> PASS / WARN / BLOCK
    |
    v
CLI report and GitHub Action result
```

**Academic ownership:** The student will implement and evaluate the code. External feedback is used for explanation, design review, debugging questions and small illustrative examples, not complete assessed modules.

## 1. Requirements and dependencies

### Required now

| Component | Why it is needed | Installation |
| --- | --- | --- |
| Python **3.11** | Runtime for the CLI and adapters | Install Python 3.11 from [python.org](https://www.python.org/downloads/) |
| Git | Version control | [git-scm.com](https://git-scm.com/downloads) |
| `venv` and `pip` | Isolated Python environment and package installation | Included with typical Python 3.11 installations |
| `pytest` | Test SARIF parsing, mapping and error handling | `python -m pip install pytest` |
| `ruff` | Lint and format checks | `python -m pip install ruff` |

**The first Semgrep SARIF reader needs no third-party runtime parsing library:** Python's standard `json`, `pathlib`, `dataclasses` and `typing` modules are sufficient for an initial implementation.

### Add only when those features are built

| Dependency/tool | Planned use |
| --- | --- |
| `PyYAML` | Read `policy.yml` later; use `yaml.safe_load`, not unsafe object deserialisation |
| `setuptools` (build backend) | Package the project through a future `pyproject.toml` |
| `build` | Optional wheel/source distribution build check |
| Semgrep CLI | Generate genuine SARIF fixtures for integration testing; **not** required to parse an existing SARIF file |
| OSV-Scanner, Gitleaks, zizmor | Add as separate external scanner tools during their later integration stages |

Avoid installing unnecessary SDKs, web frameworks, database drivers or a full SARIF object-model package before there is a demonstrated need. Document and pin compatible dependency versions when packaging is introduced.

## 2. Set up your development environment

This repository has **no `pyproject.toml` or installable package yet**. The following commands prepare your local environment; they do not install an OpenSecGate CLI that has not been written.

**Linux / macOS:**

```bash
git clone https://github.com/ymmiah/OpenSecGate.git
cd OpenSecGate
python3.11 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install pytest ruff
python --version
```

**Windows PowerShell:**

```powershell
git clone https://github.com/ymmiah/OpenSecGate.git
cd OpenSecGate
py -3.11 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install pytest ruff
python --version
```

The final version check should report Python **3.11.x**. Do not commit the `.venv/` directory. Add it to `.gitignore` when creating the initial project files.

When the student has written `pyproject.toml` and declared development extras, switch from manual tool installation to an editable development install such as `python -m pip install -e ".[dev]"`.

## 3. First development milestone: Semgrep SARIF adapter

**Goal:** Accept a saved Semgrep **SARIF 2.1.0 JSON** report and produce a collection of normalised findings. Do not add scoring or invoke external scanners in this milestone.

### Understand the input before coding

Inspect a genuine Semgrep report. Relevant SARIF fields include:

| SARIF location | Information to consider |
| --- | --- |
| `version` | SARIF format version; initially target `2.1.0` |
| `runs[]` | One or more analysis runs |
| `runs[].results[]` | Reported findings |
| `result.ruleId` | Source scanner rule identifier |
| `result.message.text` | Human-readable explanation, when provided |
| `result.level` | SARIF level: `error`, `warning`, `note` or `none` (not automatically a CVSS score) |
| `result.locations[].physicalLocation.artifactLocation.uri` | Reported file URI, when available |
| `result.locations[].physicalLocation.region.startLine` | Source line number, when available |
| `result.partialFingerprints` | Optional finding identity data |
| `runs[].tool.driver.rules[]` | Rule metadata that may be needed to enrich findings |

SARIF is a general interchange standard, **not** an OpenSecGate severity or risk-scoring policy. The mapping from SARIF levels and optional rule metadata to your internal severity categories must be explicitly designed and tested. Do not assume that optional SARIF fields always exist, or that the first run and first location are the only ones.

### Design the common finding contract first

Before writing a parser, decide the types and required/optional status of at least:

- `scanner` and `rule_id`
- `message` and `severity`
- `file_path` and `start_line` (where available)
- `fingerprint` or another stable finding identifier
- `metadata` for source-specific details that must not be silently discarded

A possible *interface to discuss*, **not a completed implementation**, is `parse_semgrep_sarif(document) -> list[Finding]`. Decide separately where JSON loading, validation and conversion belong. Preserve useful evidence rather than manufacturing missing values.

### Suggested repository layout — not yet created

```text
OpenSecGate/
├── README.md
├── .gitignore                      # to create
├── pyproject.toml                  # to create
├── policy.yml                      # later
├── src/
│   └── opensecgate/
│       ├── __init__.py
│       ├── models.py               # common Finding contract
│       └── adapters/
│           ├── __init__.py
│           └── semgrep.py          # first adapter
├── tests/
│   ├── fixtures/
│   │   └── semgrep/
│   │       └── sample.sarif        # genuine, sanitised SARIF fixture
│   └── test_semgrep_adapter.py
└── action.yml                      # later: GitHub Action metadata
```

Use this as a structure proposal, not as evidence of completed code.

### Write tests for the behaviour you intend

1. Valid SARIF containing one result produces one normalised finding.
2. A valid report with an empty `results` array produces no findings.
3. Multiple runs and multiple findings are processed without loss.
4. Missing optional location, line number or fingerprint data is handled deliberately.
5. An invalid JSON document, unsupported SARIF version or malformed required structure generates a clear error.
6. Severity mapping and any rule metadata lookups behave consistently.
7. Invalid input **never silently becomes a clean PASS**.

Use `pytest` to drive your own implementation. Once tests exist, the intended checks are:

```bash
python -m pytest -q
python -m ruff check .
```

These checks **cannot yet pass as an implementation verification**: no tests or Python source files exist in the repository.

## 4. First tasks, in order

- [ ] **Task 1 — Environment:** Clone the repository, create the Python 3.11 virtual environment, and install `pytest` and `ruff`.
- [ ] **Task 2 — Project configuration:** Write `.gitignore` and `pyproject.toml` yourself. Specify Python 3.11, a `src/` package layout, a build backend and development dependencies.
- [ ] **Task 3 — SARIF evidence:** Obtain a genuine Semgrep SARIF 2.1.0 sample and inspect its `runs`, `results` and location objects. Remove secrets and personal paths before committing the fixture.
- [ ] **Task 4 — Finding design:** Record which common fields are mandatory, which are optional and why.
- [ ] **Task 5 — Test-first adapter:** Write the first one-finding test, then implement only enough SARIF reading and mapping to make it pass.
- [ ] **Task 6 — Edge cases:** Add empty, multi-result, missing-field and malformed-input tests. Make error handling explicit.
- [ ] **Task 7 — Review:** Run tests and linting, review data loss and error behaviour, and document results before moving to the second scanner.

**Definition of done for Milestone 1:** Student-authored code can read real Semgrep SARIF, produce documented common findings, reject invalid input clearly, and pass repeatable tests. No weighted risk policy or PASS/WARN/BLOCK engine is claimed yet.

## 5. Tutor/code-review questions

Before implementing the adapter, be able to explain:

1. Why does `runs[0]["results"]` risk dropping findings?
2. What should happen when `artifactLocation` or `startLine` is missing?
3. Is SARIF `level: "error"` equivalent to an OpenSecGate `critical` finding? Why or why not?
4. Which missing fields make a report invalid, and which make only an individual field unavailable?
5. What is the difference between **zero findings** and **a scanner report that failed to load**?
6. How will you prove the adapter is correct without depending on a live Semgrep scan in every unit test?

## 6. Later milestones (not yet implemented)

- **Milestone 2:** Add and test OSV-Scanner, Gitleaks and zizmor adapters against the same finding contract.
- **Milestone 3:** Define, document and measure the six context signals; distinguish observed evidence from unavailable context.
- **Milestone 4:** Implement and validate the weighted `policy.yml` decision logic, including PASS/WARN/BLOCK thresholds and reproducibility.
- **Milestone 5:** Add the CLI and GitHub Action wrapper; document exit codes, permissions, reports and scanner-failure handling.
- **Milestone 6:** Evaluate the approach with repeatable test cases, comparisons against baseline severity-only prioritisation, limitations and a defensible academic methodology.

The GitHub Action must use least-privilege permissions and must not report PASS when a required scan or policy evaluation has failed.

## References

- [OASIS SARIF 2.1.0 specification](https://docs.oasis-open.org/sarif/sarif/v2.1.0/sarif-v2.1.0.html)
- [Semgrep documentation — local and CLI scans](https://semgrep.dev/docs/category/local-and-cli-scans)
- [Python Packaging User Guide — writing pyproject.toml](https://packaging.python.org/en/latest/guides/writing-pyproject-toml/)
- [pytest — good integration practices](https://docs.pytest.org/en/stable/explanation/goodpractices.html)
- [Ruff documentation](https://docs.astral.sh/ruff/)
- [GitHub Actions documentation](https://docs.github.com/en/actions)

---

**Implementation status:** README and development plan only. All milestones remain pending until code and tests are committed and verified.
