# Research Log — CI/CD Carbon Emissions Study
## *Towards Greener Pipelines: Measuring and Optimising Carbon Emissions in CI/CD Workflows*

**Subject project:** HTTPie CLI (`httpie/cli`)
**Measurement tool:** Eco-CI Energy Estimation v5 (`green-coding-solutions/eco-ci-energy-estimation@v5`)
**Statistical framework:** Wilcoxon signed-rank test · Bonferroni correction (α = 0.017) · Cliff's delta
**Carbon metric:** Software Carbon Intensity (SCI) — ISO/IEC 21031:2024

---

## Experiment Configuration Overview

| Config | Branch | Description |
|--------|--------|-------------|
| **C1** | `experiment/c1-baseline` | 3 separate workflow files (tests, code-style, coverage) + Eco-CI; no caching |
| **C2** | `experiment/c2-pip-cache` | Same 3 workflows + `cache: pip` on all `setup-python` steps |
| **C3** | `experiment/c3-consolidation` | All 3 stages merged into single `ci-consolidated.yml`; no caching |
| **C4** | `experiment/c4-combined` | C3 + `cache: pip` + path-based trigger filters |

Each configuration targets 30 repeated `workflow_dispatch` runs for statistical adequacy (Wilcoxon requires n ≥ 6; 30 runs provides >95% power for medium effect sizes at α = 0.017).

---

## Log Entries

---

### [2026-03-29] — Session 1: Initial Audit & Branch Remediation

#### 1.1 Scope of Audit

Following an initial code review across all four experiment branches, a systematic audit was conducted to assess the integrity of each branch prior to any data collection. The audit examined: (a) correctness of GitHub Actions YAML syntax; (b) presence and placement of Eco-CI measurement steps; (c) availability of `workflow_dispatch` triggers; (d) consistency of artifact upload naming; and (e) presence of requisite source code for CI commands to execute successfully.

#### 1.2 Issues Identified — C1 (`experiment/c1-baseline`)

The C1 branch contains the full HTTPie CLI source tree (merged from `httpie/cli`) alongside three instrumented workflow files. The following defects were identified:

**Bug C1-01 (Critical) — `continue-on-error` misplaced inside `with:` block (`tests.yml`)**
In `tests.yml`, the `Eco-CI Start Measurement` step contains `continue-on-error: true` as a key under `with:`, rather than as a top-level step attribute. In YAML terms, this means the key is passed as an input parameter to the GitHub Action (which silently ignores it) rather than being interpreted by the Actions runner as a step-level fault-tolerance directive. The practical consequence is that if the Eco-CI action fails at start-measurement (e.g., due to an upstream API issue or runner misconfiguration), the entire workflow job fails rather than continuing gracefully. All other Eco-CI steps in `tests.yml` correctly place `continue-on-error: true` at the step level.

*Fix:* Promote `continue-on-error: true` to step-level in the `Eco-CI Start Measurement` step of `tests.yml`.

**Bug C1-02 (High) — `pyopenssl` matrix dimension causing artifact name collisions (`tests.yml`)**
The test job matrix includes `pyopenssl: [0, 1]`, doubling the number of jobs from 3 to 6 (3 Python versions × 2 pyopenssl values). The artifact upload step uses the naming pattern `eco-ci-results-tests-py${{ matrix.python-version }}-${{ matrix.os }}`, which omits the `pyopenssl` dimension. Consequently, two jobs sharing the same Python version (e.g., 3.12) will both attempt to upload an artifact named `eco-ci-results-tests-py3.12-ubuntu-latest`, with the second upload overwriting the first. This results in data loss per run and renders the energy measurements for one pyopenssl variant unrecoverable. Additionally, the pyopenssl matrix dimension is irrelevant to energy consumption measurement; its presence represents an upstream testing concern that was not removed when the workflow was adapted for research purposes.

*Fix:* Remove the `pyopenssl` matrix dimension entirely. Testing with and without PyOpenSSL is not a variable of interest in the energy study.

**Bug C1-03 (Medium) — `ci-consolidated.yml` present on C1 branch**
The C1 branch includes `ci-consolidated.yml`, the workflow file designed for C3/C4 configurations. This file has a `workflow_dispatch` trigger, meaning it can be manually triggered from the GitHub Actions UI while on the `c1-baseline` branch. If triggered accidentally, it would produce energy measurements structurally equivalent to C3 (no caching, consolidated jobs) while being attributed to the C1 branch by the `collect_results.py` script. This would silently corrupt the C1 dataset.

*Fix:* Remove `ci-consolidated.yml` from the C1 branch.

**Note C1-04 (Low) — Upstream non-research workflows present**
The C1 branch retains all original HTTPie upstream workflow files: `benchmark.yml`, `content.yml`, `docs-check-markdown.yml`, `docs-deploy.yml`, `release-*.yml`, `stale.yml`, `test-package-*.yml`. While most are scoped to `push: branches: [master]` and will not trigger on `experiment/**` branches, their presence introduces ambiguity in the GitHub Actions UI and represents unnecessary surface area. These are removed as a precautionary measure to ensure clean data collection.

*Fix:* Remove all non-research workflow files from C1.

---

#### 1.3 Issues Identified — C2 (`experiment/c2-pip-cache`)

The C2 branch was created with the intent of adding `cache: pip` to all `setup-python` steps. The branch was constructed by merging changes on top of the httpie main source, but the merge was not fully resolved, leaving all three workflow files in a broken state.

**Bug C2-01 (Critical) — Unresolved merge conflicts in all three workflow files**
`tests.yml`, `code-style.yml`, and `coverage.yml` all contain unresolved Git merge conflict markers (`<<<<<<< HEAD`, `=======`, `>>>>>>> main`). GitHub Actions YAML parsing is strict: the presence of these markers renders the files syntactically invalid. No workflow on C2 will parse or execute in its current state.

*Fix:* Rewrite all three workflow files from clean, conflict-free definitions.

**Bug C2-02 (Critical) — `workflow_dispatch` trigger absent from all three workflows**
None of the three C2 workflow files include a `workflow_dispatch` trigger. Since the research protocol requires manual triggering of 30 runs per configuration, the absence of this trigger means C2 cannot be triggered on-demand and can only run via `push` or `pull_request` events. This makes controlled data collection impossible without committing dummy changes to the branch.

*Fix:* Add `workflow_dispatch` with a `reason` input to all three C2 workflow files.

**Bug C2-03 (Critical) — Eco-CI instrumentation stripped from `code-style.yml` and `coverage.yml`**
As a consequence of the failed merge resolution, the HEAD-side content (which contained the Eco-CI instrumentation) was effectively discarded in favour of the `main`-side content (which was the original, uninstrumented HTTPie workflow). Both `code-style.yml` and `coverage.yml` on C2 currently contain no Eco-CI steps and would produce no energy measurement artifacts even if triggered.

*Fix:* Restore full Eco-CI instrumentation to `code-style.yml` and `coverage.yml`, and add `cache: pip` to their `setup-python` steps.

**Bug C2-04 (High) — Duplicate `cache: pip` key in `tests.yml`**
In the partially-merged `tests.yml`, the `cache: pip` key appears twice within the same `with:` block for `setup-python`. In YAML, duplicate keys within a mapping are technically undefined behaviour (the second value typically wins in most parsers). Regardless of parser behaviour, the presence of two identical keys is structurally invalid and would be flagged by a YAML linter.

*Fix:* Remove the duplicate `cache: pip` entry in the rewritten `tests.yml`.

**Bug C2-05 (High) — Excessively large test matrix in `tests.yml`**
The C2 `tests.yml` matrix defines `os: [ubuntu-latest, macos-13, windows-latest]`, `python-version: [3.12, 3.11, 3.10, 3.9, 3.8, 3.7]`, and `pyopenssl: [0, 1]`, yielding 3 × 6 × 2 = 36 parallel jobs per run. For the purposes of this energy study, cross-platform and multi-Python comparability is not a variable under investigation. The matrix should be restricted to `ubuntu-latest` and three Python versions (3.10, 3.11, 3.12) to match the C1 configuration and ensure like-for-like comparisons.

*Fix:* Restrict matrix to `ubuntu-latest` and `python-version: [3.12, 3.11, 3.10]` only.

**Bug C2-06 (Medium) — `ci-consolidated.yml` present on C2 branch**
Same issue as C1-03. The consolidated workflow file is present and triggerable from the C2 branch.

*Fix:* Remove `ci-consolidated.yml` and all non-research workflow files from C2.

---

#### 1.4 Issues Identified — C3 (`experiment/c3-consolidation`)

The C3 branch was created from an early repository commit (prior to the httpie source merge) and contains only the research scaffolding files. The `ci-consolidated.yml` workflow file is structurally correct for C3 (no caching, no path filters, all three stages consolidated). However, the branch is missing the entire httpie application source tree.

**Bug C3-01 (Critical) — Missing httpie source code, tests, and Makefile**
The C3 branch contains only `README.md`, `analysis/energy_analysis.ipynb`, `results/`, `scripts/collect_results.py`, and `.github/workflows/ci-consolidated.yml`. The workflow's jobs execute `make venv`, `make install`, `make codestyle`, `make test`, `make test-cover`, and `make test-dist`. None of these commands can succeed without the `Makefile`, `httpie/` source package, `tests/` directory, and `setup.cfg`/`setup.py` build configuration files. Every single CI job would fail at the first `make` invocation.

*Fix:* Cherry-pick the full httpie source tree from `experiment/c1-baseline` into C3 (selective checkout of all non-`.github` content from C1), preserving the existing C3 workflow file.

---

#### 1.5 Issues Identified — C4 (`experiment/c4-combined`)

The C4 branch was branched from C3 and therefore inherits the same missing source tree. The `ci-consolidated.yml` is correctly configured for C4: path filters are uncommented and active, and `cache: pip` is present on all `setup-python` steps.

**Bug C4-01 (Critical) — Missing httpie source code (inherited from C3)**
Same root cause as C3-01. Without the httpie source tree, all CI jobs fail immediately.

*Fix:* Same selective checkout of C1 source files applied to C4.

**Note C4-02 (Low) — Blank line within `with:` block before `cache: pip`**
In the C4 `ci-consolidated.yml`, a blank line appears between `python-version: '...'` and `cache: pip` within the `with:` mapping block. The YAML specification permits blank lines within block collections, and GitHub Actions' parser (which uses the `js-yaml` library) handles this gracefully. However, the style is inconsistent with standard GitHub Actions workflow conventions and warrants cleanup for clarity.

*Fix:* Remove blank lines within `with:` blocks in `ci-consolidated.yml` on C4.

---

#### 1.6 Remediation Actions Taken

All issues identified above were resolved in this session. The following changes were applied to each branch:

| Branch | Actions |
|--------|---------|
| `experiment/c1-baseline` | Fixed `continue-on-error` placement in `tests.yml`; removed `pyopenssl` matrix dimension; removed `ci-consolidated.yml` and all non-research workflow files |
| `experiment/c2-pip-cache` | Rewrote `tests.yml`, `code-style.yml`, `coverage.yml` from clean definitions with full Eco-CI instrumentation, `cache: pip`, `workflow_dispatch`, and correct ubuntu-only matrix; removed non-research workflows |
| `experiment/c3-consolidation` | Imported full httpie source tree from `experiment/c1-baseline`; `ci-consolidated.yml` retained as-is (already correct for C3) |
| `experiment/c4-combined` | Imported full httpie source tree from `experiment/c1-baseline`; removed blank lines within `with:` blocks in `ci-consolidated.yml` |

All changes committed and pushed to origin. Branches are now in a state where `workflow_dispatch` runs should produce valid Eco-CI energy measurement artifacts.

---

#### 1.7 Next Steps

- [ ] Trigger 30 × `workflow_dispatch` runs per branch (C1–C4) via the GitHub Actions UI
- [ ] Verify at least one run per branch produces Eco-CI artifacts before committing to the full 30-run protocol
- [ ] Run `scripts/collect_results.py` after all runs complete
- [ ] Execute `analysis/energy_analysis.ipynb` to produce statistical results
- [ ] Populate Sections 4 and 5 of `paper/paper-draft.md` with quantitative findings

---

*Log maintained by: research author*
*Last updated: 2026-03-29*
