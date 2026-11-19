# OpenSecGate — Development Logbook

This logbook records the work completed on OpenSecGate, the decisions made during development, problems identified and what needs to happen next. Entries are dated and linked to repository changes where possible. Planned work is kept separate from work that has actually been completed.

The project follows the six-stage development roadmap in the [README](../README.md). The planned code-freeze date is **12 February 2027**; evaluation follows the code freeze.

## Development progress

| Stage | Description | Status |
| --- | --- | --- |
| 1 | Setup and common finding format | **In progress** — project structure prepared; finding schema and tests outstanding |
| 2 | Scanner integration | Not started |
| 3 | Context enrichment | Not started |
| 4 | Decision engine | Not started |
| 5 | GitHub Action | Not started |
| 6 | Evaluation | Not started |

---

## 19 November 2026 — Initial project setup and development plan

**Stage:** 1 — Setup and common format  
**Focus:** Confirm the project scope, prepare the repository and agree the order of development.

### Work carried out

Today I reviewed the OpenSecGate interim project report and used it to confirm what the tool is meant to do. The project will use Semgrep, OSV-Scanner, Gitleaks and zizmor to collect security findings. OpenSecGate will then convert those results into one format, add six context signals and apply the rules in `policy.yml` to return PASS, WARN or BLOCK.

I also confirmed that this is a Python 3.11 command-line project which will later be packaged as a GitHub Action. The scope remains npm and PyPI projects running through GitHub Actions. It will not include a web interface or machine-learning-based decisions.

The repository was prepared for the first development stage. The README was expanded to describe the architecture, scope, required tools, development stages and evaluation plan. It was then revised to keep the public documentation focused on the project itself.

The following starting files and folders were added to the repository:

| File or folder | Purpose |
| --- | --- |
| `pyproject.toml` | Defines the Python 3.11 package, setuptools build configuration and optional development dependencies (`pytest` and `ruff`) |
| `.gitignore` | Excludes local virtual environments, Python caches, build outputs and local configuration files |
| `src/opensecgate/__init__.py` | Creates the initial Python package |
| `src/opensecgate/adapters/__init__.py` | Prepares the package for individual scanner adapters |
| `tests/` | Provides a location for tests and scanner-output fixtures |

The six development stages were agreed in order: setup and common format; scanner integration; context enrichment; decision engine; GitHub Action; and evaluation.

### Decisions and reasons

The first adapter will handle Semgrep's **SARIF 2.1.0** output. This gives the project a clear starting point without needing to integrate all four scanners at once.

The normalised finding format will be defined before the adapter logic. This should make it easier for the other three adapters to return information in the same structure.

The first parser can use Python's standard `json` module, so there is no need to add a third-party SARIF library at this stage. `pytest` and `ruff` are configured as development dependencies. Other packages will be introduced when the related features are ready.

### Problems, risks and limitations

The main issue at this stage is making sure findings from different tools can be represented consistently without losing useful evidence. SARIF reports may also omit optional fields, so the adapter will need to handle these cases safely.

It is important to distinguish between **a valid scan with zero findings** and **a scanner report that could not be read**. An invalid report must not produce a clean PASS.

The repository structure has been committed, but there is **no implemented finding model, scanner adapter, policy engine or executable GitHub Action yet**. No local installation, unit-test run or successful build has been verified for this entry.

### Next work

1. Write and review the common `Finding` schema, including required and optional fields.
2. Create the first tests for that schema.
3. Obtain a genuine Semgrep SARIF sample with sensitive details removed.
4. Implement and test the Semgrep adapter incrementally.
5. Record the results, problems and relevant commits in the next logbook entry.

### Evidence of progress

The following Git commits record the documentation and setup changes made today:

- [`0c3ca249`](https://github.com/ymmiah/OpenSecGate/commit/0c3ca249a8c6e1452f3a9df83dc061fb310cbfa1) — Initial README development plan.
- [`b125db0c`](https://github.com/ymmiah/OpenSecGate/commit/b125db0c0e) — README wording revised for public documentation.
- [`657e37a7`](https://github.com/ymmiah/OpenSecGate/commit/657e37a768) — README aligned with the interim report.
- [`5428c7ee`](https://github.com/ymmiah/OpenSecGate/commit/5428c7ee21ff7cd514c3ee7233b273057e30aca2) — `.gitignore` added.
- [`4c2763c0`](https://github.com/ymmiah/OpenSecGate/commit/4c2763c07b0b2f92ff145204e7b832bbe6d0fcd4) — Python package configuration added.
- [`567a63b6`](https://github.com/ymmiah/OpenSecGate/commit/567a63b62ef0ad07ed462927aa4405020b82b777) and [`e3dcdfb2`](https://github.com/ymmiah/OpenSecGate/commit/e3dcdfb2dda2c489ea4429b7a4ae4ae2e4ca4869) — Initial source packages created.
- [`fac0a1fe`](https://github.com/ymmiah/OpenSecGate/commit/fac0a1feda056153f3cd77bd3d6bc931ab98d019) — Test directory prepared.
- [`ae0ac767`](https://github.com/ymmiah/OpenSecGate/commit/ae0ac767cf1ed17831668d35adc66049639cc187) — README updated to reflect the repository setup.

**End-of-day position:** Stage 1 has started. The repository now has its basic Python structure and documentation. The normalised finding schema and its tests are the next items to complete.

---

## Logbook update format

New dated entries should record the **stage**, **work completed**, **decisions and reasons**, **problems or limitations**, **tests and evidence**, and **next steps**. Only confirmed results should be marked as complete. Where a task continues over several days, record the progress without describing it as finished.
