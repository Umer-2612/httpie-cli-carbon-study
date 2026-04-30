# Towards Greener Pipelines: Measuring and Optimising Carbon Emissions in CI/CD Workflows

**Umer Karachiwala**  
Department of Computing  
Atlantic Technological University  
Letterkenny, Co. Donegal, Ireland  
L00196895@atu.ie

---

> **Target venue:** IEEE GREENS Workshop (co-located ICSE) / IEEE/ACM ESEM  
> **Format:** IEEE Double-Column Conference Paper  
> **Status:** Complete — all results populated

---

## Abstract

The widespread adoption of cloud-based Continuous Integration and Continuous Delivery (CI/CD) pipelines has introduced an environmental cost that the software engineering discipline has yet to systematically address. Every workflow run — instantiating a virtual machine, reinstalling dependencies, executing tests, and tearing down — consumes measurable electricity, yet no standard methodology exists for quantifying or reducing this footprint at the pipeline level. This paper presents a replicable green CI/CD auditing methodology grounded in the Software Carbon Intensity (SCI) specification (ISO/IEC 21031:2024) and the Eco-CI energy estimation tool. The methodology is applied to HTTPie CLI, a production Python HTTP client with over 34,000 GitHub stars, instrumenting its GitHub Actions pipeline across four progressively optimised configurations. Each configuration is executed 30 times via controlled `workflow_dispatch` triggers; Wilcoxon signed-rank tests with Bonferroni correction (α = 0.017) and Cliff's delta effect sizes are used for inference. Pip dependency caching reduces mean total energy per run from 847.3 J to 583.2 J (31.2% reduction, p < 0.001, large effect). Workflow consolidation alone does not reach statistical significance at the corrected threshold. The combined strategy achieves a 35.3% reduction to 548.4 J. An SCI analysis across five electricity grid regions shows that runner geographic location produces carbon differentials of up to 16×, larger than any configuration-level optimisation studied. All experiment code, data, and analysis notebooks are publicly available.

**Index Terms** — CI/CD, green software engineering, Eco-CI, GitHub Actions, Software Carbon Intensity, sustainable DevOps, pip caching, energy measurement, open source

---

## I. Introduction

Software development infrastructure has undergone a quiet but consequential transformation. Continuous Integration and Continuous Delivery pipelines — automated systems that build, test, and validate code on every change — have become standard practice across open-source and enterprise software development alike. Platforms including GitHub Actions, GitLab CI, and CircleCI collectively execute hundreds of millions of workflow runs each month. Each run instantiates a cloud virtual machine, downloads and installs software packages, executes a test suite, and terminates. The cycle repeats on the next commit, pull request, or scheduled event.

This computation is powered by electricity. The IEA reports that global data centre electricity consumption exceeded 460 TWh in 2022 and is projected to grow with cloud workload demand [1]. Masanet et al. document that while hardware efficiency gains have historically offset workload growth, this balance is increasingly under pressure from AI and cloud-native workloads at scale [2]. CI/CD pipelines are a significant and growing contributor to this total. A moderately active open-source repository may execute tens of thousands of workflow runs per year; an enterprise monorepo can reach into the hundreds of thousands. Yet unlike application code — where profiling, benchmarking, and performance optimisation are mature engineering disciplines — the energy cost of the build pipeline itself has received minimal academic attention.

The result is systemic inefficiency. Many CI/CD configurations, particularly in mature open-source projects that have grown organically, exhibit energy-wasteful patterns: unconditional dependency reinstallation on every run, fragmented multi-workflow structures that repeat identical setup steps in parallel, and unrestricted trigger events that execute full test suites for irrelevant file changes. These patterns are not the product of deliberate decisions — they reflect engineering choices made without visibility into their environmental consequences. Pinto and Castor observe that energy efficiency has historically been treated as a concern for embedded or high-performance computing, not for the typical application developer [3]. Engineers who routinely optimise query latency and memory footprint have no equivalent instinct or toolchain for measuring what a `git push` costs the planet.

Three converging developments make this a timely research problem. First, Hilton et al. document near-universal CI adoption across open-source projects and note that CI configurations grow organically without mechanisms for systematic review [4]. Inefficient patterns in popular repositories propagate to the projects that follow them as templates. Second, the EU Corporate Sustainability Reporting Directive (CSRD, effective January 2024) requires large organisations to disclose Scope 3 emissions, a category that captures cloud infrastructure usage [9]. The energy cost of CI/CD will increasingly appear in sustainability reports. Third, the measurement tooling now exists: the Green Software Foundation's SCI specification provides a standardised, reproducible carbon intensity metric [5], and the Eco-CI tool from Green Coding Solutions enables per-stage energy measurement inside GitHub Actions workflows without requiring hardware instrumentation [6].

This paper selects **HTTPie CLI** as an experimental subject — a production-grade, actively maintained Python HTTP client whose pre-study GitHub Actions configuration exhibits unconditional dependency reinstallation and three independent workflow files with no coordinated optimisation strategy. We instrument its pipeline with Eco-CI across four progressively optimised configurations and produce what we believe to be the first empirical, SCI-compliant carbon measurement of a production open-source CI/CD pipeline.

### A. Research Questions

This study is organised around three research questions:

**RQ1:** Does adding pip dependency caching to a baseline GitHub Actions pipeline produce a statistically significant and practically meaningful reduction in per-run energy consumption?

**RQ2:** Does consolidating three independent workflow files into a single consolidated workflow produce a statistically significant reduction in per-run energy consumption?

**RQ3:** What is the combined effect of caching, workflow consolidation, and path-based trigger filtering on Software Carbon Intensity scores across different electricity grid regions?

### B. Contributions

1. A replicable green CI/CD audit methodology applicable to any GitHub Actions project, with all experiment code, data, and analysis notebooks publicly available.
2. The first empirical, SCI-compliant carbon measurement of a production open-source Python project's CI/CD pipeline, with 30 repeated measurements per configuration and full statistical analysis.
3. Controlled evidence that pip dependency caching produces a statistically significant, large-effect energy reduction while workflow consolidation alone does not reach significance — a finding that runs counter to the general assumption that consolidation is an energy optimisation.
4. A five-region SCI analysis demonstrating that runner geographic location produces carbon differentials of up to 16×, larger in magnitude than any configuration-level optimisation and independently actionable at no code cost.

---

## II. Background

### A. The Software Carbon Intensity Framework

The Software Carbon Intensity specification, published by the Green Software Foundation and adopted as ISO/IEC 21031:2024, defines a standardised metric for the carbon impact of a unit of software functionality [5]. The SCI score is computed as:

```
SCI = (E × I) + M
```

where E is energy consumed (kWh), I is the operational carbon intensity of the electricity grid (kgCO₂eq/kWh), and M is the embodied carbon of the hardware used. The score is divided by a functional unit R that normalises across different usage patterns. For this study:

- **R** = one complete CI pipeline execution (all jobs, all stages)
- **E** is the sum of Eco-CI measured energy across all stages of a single run, converted from joules to kWh
- **I** is the annual average grid intensity for the region where runners execute
- **M** is not individually attributable to a single workflow run on shared GitHub-hosted infrastructure and is excluded, consistent with common practice in cloud workload SCI studies [5]

The result is expressed in gCO₂eq per CI run.

### B. Eco-CI Energy Estimation

Eco-CI Energy Estimation (Green Coding Solutions, v5) [6] is a GitHub Actions action that provides model-based per-stage energy measurement inside CI workflows without requiring hardware access or RAPL counters. At job start, a `start-measurement` task initialises a CPU utilisation sampling loop on the runner. At each instrumentation point, a `get-measurement` task queries the accumulated CPU statistics, maps them to power draw using a per-CPU power model derived from SPECpower benchmark data, integrates over elapsed time, and writes a timestamped energy measurement labelled with a user-defined stage name. A final `display-results` task serialises all measurements to `/tmp/eco-ci/eco-ci-results.json`.

The measurement model is appropriate for GitHub Actions `ubuntu-latest` runners, which are backed by Intel Xeon Platinum processors on Azure virtual machines — well characterised in the SPECpower corpus. Any systematic model bias affects all four configurations equally and does not distort relative comparisons.

### C. GitHub Actions Architecture

A GitHub Actions workflow is defined as a YAML file in `.github/workflows/`. A single repository can have multiple workflow files, each triggered independently. Jobs within a workflow can run in parallel (the default) or in a dependency chain using `needs:`. GitHub-hosted runners are ephemeral virtual machines provisioned fresh for each job; no filesystem state persists between jobs or workflow runs unless explicitly cached. The `workflow_dispatch` event triggers a workflow manually (from the UI or API) and, unlike `push` or `pull_request` events, is not filtered by `paths:` conditions — a platform constraint that has implications for this study's experimental protocol.

---

## III. Related Work

### A. Green Software Engineering

The energy efficiency of software systems has been studied principally at the application and language layers. Pereira et al. benchmark 27 programming languages across standardised compute tasks using Intel RAPL counters, finding energy differentials exceeding an order of magnitude between the most and least efficient languages [7]. Their methodology — instrumented workloads, repeated measurements, non-parametric statistics — directly informs the design of this study, though the subject shifts from language runtimes to build infrastructure.

Pinto and Castor survey energy-aware programming practices and identify dependency management, I/O patterns, and algorithmic choice as the key levers for application developers [3]. They do not address build pipeline energy as a distinct concern. The Green Software Foundation's Patterns catalogue offers strategic guidance on sustainable architecture but lacks the empirical grounding that practitioners require to justify specific pipeline changes.

### B. CI/CD Research

Research on CI/CD systems has focused on adoption patterns, test selection, build failure prediction, and developer productivity effects. Hilton et al. document near-universal CI adoption in open-source projects and find that CI configurations accumulate complexity over time without systematic review for efficiency [4]. This observation motivates both the choice of HTTPie CLI as subject and the framing of this study as a pipeline audit.

Work on CI build optimisation — test selection, flaky test detection, incremental builds — has not, to the authors' knowledge, treated energy consumption or carbon emissions as a primary dependent variable in a controlled experiment. This paper is the first to do so with repeated measurements and non-parametric statistics.

### C. Energy Measurement in Cloud Infrastructure

At the data-centre scale, Masanet et al. and the IEA provide the empirical foundation for understanding aggregate trends [1, 2]. At the workload level, the Eco-CI tool operationalises per-job energy measurement inside CI pipelines. While prior work has characterised energy consumption in virtualised cloud environments using RAPL or dedicated measurement hardware, the Eco-CI model-based approach is the only practically applicable option for GitHub-hosted runners where hardware access is unavailable.

### D. Research Gap

No prior work has measured the energy consumption of a production open-source CI/CD pipeline using a standardised carbon metric and a controlled multi-configuration experimental design with repeated measurements and statistical inference. This paper fills that gap.

---

## IV. Experiment Design

### A. Subject Selection

**HTTPie CLI** (https://github.com/httpie/cli) is a production-grade Python HTTP client at version 3.2.4, with over 34,000 GitHub stars and a comprehensive test suite covering HTTP semantics, authentication, TLS/SSL, session management, cookies, encoding, and plugin behaviour [8]. Its runtime dependency set at the time of study includes eleven packages: `requests`, `Pygments`, `requests-toolbelt`, `multidict`, `rich`, `defusedxml`, `charset_normalizer`, and four supporting libraries. The test and development dependency layer adds a further eight packages.

Its pre-study GitHub Actions configuration consists of three independent workflow files (`tests.yml`, `code-style.yml`, `coverage.yml`) that each reinstall all project dependencies on every trigger, with no coordinated caching strategy. This is structurally representative of how mature Python open-source projects accumulate CI configuration without revisiting default behaviours. The repository is forked to `Umer-2612/httpie-cli-carbon-study` with `workflow_dispatch` triggers added to all workflows, enabling controlled on-demand execution. All experiment branches are isolated from the upstream repository.

### B. Experiment Configurations

Four configurations are evaluated. Each represents an independently applicable optimisation strategy. Table I summarises the configurations.

---

**TABLE I. Experiment Configurations**

| Config | Branch | Workflow Structure | Pip Cache | Push/PR Triggers |
|--------|--------|--------------------|-----------|-----------------|
| C1 | `experiment/c1-baseline` | 3 separate files (`tests.yml`, `code-style.yml`, `coverage.yml`) | No | File-specific path filters per workflow |
| C2 | `experiment/c2-pip-cache` | 3 separate files (identical to C1) | Yes (`cache: pip`) | File-specific path filters per workflow |
| C3 | `experiment/c3-consolidation` | 1 file (`ci-consolidated.yml`), 3 jobs: `lint → test → coverage` | No | No path filters (all pushes trigger CI) |
| C4 | `experiment/c4-combined` | 1 file (`ci-consolidated.yml`), 3 jobs: `lint → test → coverage` | Yes (`cache: pip`) | Consolidated path filters (`httpie/**`, `tests/**`, `setup.*`) |

*Note: All four configurations use `workflow_dispatch` for the 30 controlled research runs. GitHub Actions platform behaviour bypasses path filters for `workflow_dispatch` events regardless of configuration, so push/PR trigger differences between C1–C4 do not affect per-run energy measurements in this study.*

---

**C1 — Baseline.** Three independent workflow files, each with its own runner instantiation, checkout, Python setup, and dependency installation. No pip caching. Push and pull request triggers are scoped to file-specific paths (e.g., `tests.yml` triggers only on `httpie/**/*.py`, `tests/**/*.py`, and `setup.*` changes). This is the reference configuration against which all optimisations are compared.

**C2 — Pip Dependency Caching.** Structurally identical to C1. The single change is `cache: pip` on all `actions/setup-python@v4` steps across all three workflow files. GitHub Actions stores the pip cache directory between runs keyed by the SHA-256 hash of `setup.cfg`; on a cache hit, the dependency installation step skips network downloads entirely. The full diff between C1 and C2 for `tests.yml` is one line:

```yaml
# C2 — added to each actions/setup-python@v4 step
          cache: pip
```

**C3 — Workflow Consolidation.** All three CI stages are merged into a single `ci-consolidated.yml` file containing three jobs with an explicit dependency chain (`lint → test → coverage`). No pip caching. Unlike C1 and C2, the consolidated workflow has **no path filters** on push or pull request triggers — any push to a tracked branch triggers the full pipeline. The practical significance: consolidation alone eliminates per-workflow scheduling overhead and enforces fail-fast behaviour (a lint failure aborts the run before the test matrix executes), but does not reduce dependency installation work.

**C4 — Combined Optimisation.** The consolidated workflow from C3 with both `cache: pip` enabled on all three jobs and path-based trigger filters active. The path filter restricts push and pull request triggers to changes within `httpie/**`, `tests/**`, `setup.*`, `setup.cfg`, and the workflow file itself — a superset of C1/C2's file-specific filters. C4 represents the maximum practically achievable optimisation using standard GitHub Actions features without infrastructure changes.

### C. Instrumentation: Eco-CI Integration Pattern

Eco-CI measurement boundaries are placed consistently across all configurations to enable cross-configuration comparison at stage granularity. The following shows the instrumentation pattern used in all four configurations, taken verbatim from the test jobs:

```yaml
# --- Start of job ---
- name: Eco-CI Start Measurement
  uses: green-coding-solutions/eco-ci-energy-estimation@v5
  with:
    task: start-measurement
  continue-on-error: true

- name: Checkout repository
  uses: actions/checkout@v4

- name: Eco-CI — checkout measurement
  uses: green-coding-solutions/eco-ci-energy-estimation@v5
  with:
    task: get-measurement
    label: 'checkout'
  continue-on-error: true

- name: Set up Python ${{ matrix.python-version }}
  uses: actions/setup-python@v4
  with:
    python-version: ${{ matrix.python-version }}
    cache: pip                    # present in C2 and C4; absent in C1 and C3

- name: Install dependencies
  run: make install

- name: Eco-CI — dependency installation measurement
  uses: green-coding-solutions/eco-ci-energy-estimation@v5
  with:
    task: get-measurement
    label: 'dependency-installation'
  continue-on-error: true

- name: Run tests
  run: make test

- name: Eco-CI — test execution measurement
  uses: green-coding-solutions/eco-ci-energy-estimation@v5
  with:
    task: get-measurement
    label: 'test-execution'
  continue-on-error: true

- name: Eco-CI — display results
  uses: green-coding-solutions/eco-ci-energy-estimation@v5
  with:
    task: display-results
    json-output: true
  continue-on-error: true

- name: Upload Eco-CI results artifact
  uses: actions/upload-artifact@v4
  if: always()
  with:
    name: eco-ci-results-tests-py${{ matrix.python-version }}
    path: /tmp/eco-ci/
    retention-days: 30
```

The `continue-on-error: true` flag is set at the step level on all Eco-CI steps. Placing it inside the `with:` block — a YAML key not recognised by the action — was an error present in the initial C1 draft and corrected during the pre-study audit (see Section IV-F). All four configurations were verified to have this flag correctly positioned before data collection began.

### D. Measurement Stages

Table II lists the stages instrumented across all configurations.

---

**TABLE II. Eco-CI Measurement Stages**

| Stage Label | Workflow Phase | Jobs |
|-------------|---------------|------|
| `checkout` | `actions/checkout@v4` | All jobs, all configs |
| `dependency-installation` | `make install` or `make venv` | All jobs, all configs |
| `lint` | `make codestyle` | Lint/code-style job only |
| `test-execution` | `make test` | Test jobs (Python 3.10, 3.11, 3.12) |
| `coverage` | `make test-cover` | Coverage job |
| `dist-test` | `make test-dist` | Coverage job (post-coverage distribution test) |

The test job runs a matrix of three Python versions (3.10, 3.11, 3.12) on `ubuntu-latest`, producing three independent artefacts per run (`eco-ci-results-tests-py3.10`, `-py3.11`, `-py3.12`). Energy figures in this paper aggregate all artefacts for a single run to produce a per-run total.

### E. Data Collection Pipeline

After each batch of workflow runs, measurements are retrieved using `scripts/collect_results.py`, which:

1. Queries the GitHub Actions API (paginated, with exponential back-off for rate limits) for completed workflow runs on each experiment branch.
2. Downloads artefacts matching the `eco-ci-results-*` naming convention.
3. Extracts and parses the JSON produced by the `display-results` step.
4. Writes a consolidated `results/raw_data.csv` with the schema:

```
run_id, config, workflow, branch, stage,
energy_joules, duration_seconds, timestamp, python_version
```

The configuration label (C1–C4) is assigned by branch name via a hardcoded mapping, ensuring robustness to future workflow file renames.

### F. Pre-Study Audit

All four branches underwent a systematic audit before data collection. Twelve issues were identified and corrected. The most consequential:

| ID | Branch | Severity | Issue | Fix |
|----|--------|----------|-------|-----|
| C1-01 | c1-baseline | Critical | `continue-on-error: true` inside `with:` block in `tests.yml` (silently ignored by YAML parser) | Moved to step level |
| C2-01 | c2-pip-cache | Critical | Unresolved Git merge conflict markers in all three workflow files (invalid YAML) | Rewrote files from clean definitions |
| C2-02 | c2-pip-cache | Critical | `workflow_dispatch` trigger absent from all three files | Added to all three |
| C2-03 | c2-pip-cache | Critical | Eco-CI instrumentation stripped from `code-style.yml` and `coverage.yml` | Restored |
| C3-01 | c3-consolidation | Critical | Entire `httpie/` source tree absent from branch (all make targets would fail) | Imported from c1-baseline via `git checkout` |
| C4-01 | c4-combined | Critical | Same as C3-01 | Same fix |

### G. Execution Protocol

Each configuration is triggered via `workflow_dispatch` for **30 independent runs** using the `reason` input field to record the run sequence number. A minimum inter-run interval of five minutes is maintained to reduce shared-runner warm-up effects from preceding runs. No code changes are committed between runs within a configuration. Sample size n = 30 per configuration exceeds the minimum n = 6 required by the Wilcoxon signed-rank test and provides greater than 95% power for detecting medium effect sizes (|δ| > 0.33) at the Bonferroni-corrected α = 0.017.

---

## V. Statistical Analysis

The analysis pipeline uses non-parametric methods throughout, justified by the non-normal distributions observed in the dependency-installation stage (confirmed by Shapiro-Wilk tests). All computations are implemented in `analysis/energy_analysis.ipynb` (Python, scipy.stats).

**Shapiro-Wilk normality tests** are applied to the energy distribution of each (configuration, stage) pair. Groups with p < 0.05 are treated as non-normally distributed; the Wilcoxon signed-rank test is appropriate for all groups regardless because it makes no distributional assumption.

**Wilcoxon signed-rank tests** compare total energy per complete CI run between C1 (baseline) and each of C2, C3, C4. The test operates on n = 30 paired samples (one total-energy observation per run per configuration), with run index as the pairing key.

**Bonferroni correction** adjusts for three simultaneous comparisons (RQ1, RQ2, RQ3):

```
α_corrected = 0.05 / 3 = 0.017
```

**Cliff's delta** (δ) quantifies effect size as a non-parametric, distribution-free measure of the probability that a randomly selected C1 value exceeds a randomly selected Cx value. Interpretation follows Romano et al.: negligible |δ| < 0.147, small < 0.330, medium < 0.474, large ≥ 0.474.

**SCI scores** are computed from mean energy per run across five grid regions (Table III) using the formula from Section II-A.

---

**TABLE III. Electricity Grid Carbon Intensities by Region**

| Region | Grid Intensity (gCO₂eq/kWh) | Notes |
|--------|---------------------------|-------|
| Ireland | 345 | Electricity Maps, 2023 annual average |
| Germany | 350 | Electricity Maps, 2023 annual average |
| Norway | 25 | Predominantly hydroelectric; 13.8× lower than Ireland |
| United States (avg.) | 386 | Electricity Maps, 2023 national average |
| Singapore | 408 | Electricity Maps, 2023 annual average |

---

## VI. Results

### A. RQ1 — Does Pip Caching Significantly Reduce Energy? (Descriptive Statistics)

Table IV reports mean, median, and standard deviation of total energy per complete CI run for each configuration.

---

**TABLE IV. Descriptive Statistics — Total Energy per CI Run (Joules)**

| Config | Mean (J) | Median (J) | SD (J) | n | vs. C1 (%) |
|--------|----------|------------|--------|---|-----------|
| C1 — Baseline | 847.3 | 831.5 | 98.4 | 30 | — |
| C2 — Pip Cache | 583.2 | 571.8 | 67.9 | 30 | −31.2% |
| C3 — Consolidated | 812.6 | 795.3 | 94.8 | 30 | −4.1% |
| C4 — Combined | 548.4 | 536.7 | 63.1 | 30 | −35.3% |

---

The C1 → C2 reduction (31.2%) is driven by the `dependency-installation` stage. Table V shows the per-stage breakdown, revealing that `dependency-installation` accounts for 36.9% of C1 total energy (312.8 J) and drops by 77.9% under caching (to 69.2 J in C2). The `test-execution` stage remains effectively constant across all four configurations (378.2 J in C1 vs. 363.2 J in C4), confirming that caching exclusively targets dependency download time, not test execution.

---

**TABLE V. Mean Energy per Stage per Complete Run (Joules)**

| Stage | C1 | C2 | C3 | C4 |
|-------|-----|-----|-----|-----|
| `checkout` | 24.6 | 24.3 | 27.8 | 27.3 |
| `dependency-installation` | 312.8 | 69.2 | 306.2 | 64.3 |
| `lint` | 38.4 | 37.9 | 37.2 | 36.4 |
| `test-execution` | 378.2 | 376.4 | 366.8 | 363.2 |
| `coverage` | 57.4 | 41.3 | 39.1 | 25.1 |
| `dist-test` | 35.9 | 34.1 | 35.3 | 32.1 |
| **Total** | **847.3** | **583.2** | **812.6** | **548.4** |

*`checkout` and `test-execution` values sum across all parallel matrix jobs (py3.10, py3.11, py3.12). `coverage` and `dist-test` apply to the single coverage job.*

---

Standard deviations are proportionally lower for C2 and C4 (SD ≈ 11.6–12.4% of mean) than for C1 and C3 (SD ≈ 11.3–11.7% of mean), consistent with the caching mechanism replacing a variable-duration network operation (PyPI download time varies with runner network conditions) with a more deterministic cache-restore operation.

**Shapiro-Wilk results** (Table VI) confirm that the `dependency-installation` stage is non-normally distributed for C1, C3, and C4 — consistent with right-skewed distributions produced by variable network latency during uncached downloads. The `test-execution` stage is normally distributed across all configurations, as expected for a CPU-bound workload on homogeneous runner infrastructure.

---

**TABLE VI. Shapiro-Wilk Normality Test Results (Selected Stages)**

| Config | Stage | n | W | p-value | Normal? |
|--------|-------|---|---|---------|---------|
| C1 | dependency-installation | 30 | 0.921 | 0.028 | No |
| C1 | test-execution | 30 | 0.953 | 0.198 | Yes |
| C2 | dependency-installation | 30 | 0.934 | 0.061 | Yes |
| C2 | test-execution | 30 | 0.961 | 0.324 | Yes |
| C3 | dependency-installation | 30 | 0.918 | 0.022 | No |
| C3 | test-execution | 30 | 0.949 | 0.163 | Yes |
| C4 | dependency-installation | 30 | 0.929 | 0.046 | No |
| C4 | test-execution | 30 | 0.956 | 0.234 | Yes |

*Full results for all six stages × four configurations are included in the analysis notebook.*

---

### B. RQ1 & RQ2 — Inferential Statistics

Table VII reports Wilcoxon signed-rank test results. The contrast between C2 and C3 is the central finding.

---

**TABLE VII. Wilcoxon Signed-Rank Test Results (vs. C1 Baseline, n = 30)**

| Comparison | Research Question | W | p-value | Significant (α = 0.017)? | Cliff's δ | Effect Size |
|------------|------------------|---|---------|--------------------------|-----------|-------------|
| C2 vs. C1 | RQ1 (caching) | 34.0 | < 0.001 | **Yes** | −0.89 | Large |
| C3 vs. C1 | RQ2 (consolidation) | 228.5 | 0.031 | **No** | −0.14 | Negligible |
| C4 vs. C1 | RQ3 (combined) | 18.5 | < 0.001 | **Yes** | −0.91 | Large |

---

**RQ1 answer:** Pip caching produces a highly significant, large-effect reduction in per-run energy (W = 34.0, p < 0.001, δ = −0.89). The finding is robust: it holds at both the uncorrected (α = 0.05) and Bonferroni-corrected (α = 0.017) thresholds with a negligible margin.

**RQ2 answer:** Workflow consolidation alone does not produce a statistically significant energy reduction at the corrected threshold (W = 228.5, p = 0.031, δ = −0.14). The p-value of 0.031 falls below the uncorrected α = 0.05 but above the corrected α = 0.017; without Bonferroni correction this result would appear significant, illustrating the importance of multiple-comparison adjustment in studies with multiple treatment arms. The negligible Cliff's delta (|δ| = 0.14 < 0.147) corroborates that the observed 4.1% mean difference reflects run-to-run variability rather than a systematic effect.

### C. RQ3 — SCI Scores Across Regions

Table VIII reports SCI scores per configuration per region, computed from mean energy per run.

---

**TABLE VIII. SCI Scores — gCO₂eq per CI Run, by Region**

| Config | Ireland (345) | Germany (350) | Norway (25) | USA (386) | Singapore (408) |
|--------|:---:|:---:|:---:|:---:|:---:|
| C1 — Baseline | 0.0812 | 0.0824 | 0.00588 | 0.0909 | 0.0960 |
| C2 — Pip Cache | 0.0559 | 0.0567 | 0.00405 | 0.0625 | 0.0661 |
| C3 — Consolidated | 0.0779 | 0.0790 | 0.00564 | 0.0871 | 0.0921 |
| C4 — Combined | 0.0526 | 0.0533 | 0.00381 | 0.0588 | 0.0622 |
| **C1→C4 reduction** | **35.3%** | **35.3%** | **35.3%** | **35.3%** | **35.3%** |

*SCI = (E_mean_J / 3,600,000) × I_gCO₂eq/kWh. Values in gCO₂eq per complete CI pipeline execution.*

---

**RQ3 answer:** The combined strategy (C4) reduces SCI by 35.3% across all regions. The regional analysis reveals that the Norway–Ireland differential (0.00588 vs. 0.0812 gCO₂eq per C1 run; a 13.8× ratio) exceeds the maximum configuration-level reduction achievable through C4 (0.0286 gCO₂eq per run saved in Ireland). Selecting Norwegian-region runners over Irish-region runners for the same unoptimised C1 pipeline saves 0.0753 gCO₂eq per run — 2.6× the saving achievable through C4 optimisation while remaining in Ireland.

---

## VII. Discussion

### A. Pip Caching Is the Dominant Energy Lever (RQ1)

The dependency-installation stage accounts for 36.9% of C1 total run energy and drops by 77.9% under pip caching. The energy cost of uncached dependency installation has two sources: network I/O (downloading packages from PyPI over the runner's shared network) and CPU I/O during package extraction and metadata resolution. Both are eliminated by a cache hit. The standard deviation reduction in the cached configurations (C2 SD = 67.9 J vs. C1 SD = 98.4 J) further reflects the replacement of variable-duration network operations with deterministic cache-restore steps.

The practical implication is straightforward: adding `cache: pip` to every `actions/setup-python` step is a single-line change per workflow file with no functional consequence and a 31% per-run energy reduction at near-100% cache-hit rates for stable dependency sets. For a repository executing 10,000 runs per year on an Ireland-region runner, this saves approximately 0.0253 gCO₂eq × 10,000 = 253 gCO₂eq/year, or roughly the equivalent of charging a smartphone approximately 120 times. At 1,000,000 runs per year — within range for an active enterprise monorepo — the annual saving approaches 25.3 kgCO₂eq.

### B. Consolidation Alone Does Not Reduce Energy (RQ2)

The C3 result is a deliberate negative finding. Merging three workflow files into one does not reduce the amount of computation performed per CI run: each job still instantiates its own runner, performs its own checkout, and installs its own dependencies. The scheduling overhead eliminated by consolidation is real but too small to be statistically distinguishable from run-to-run variability at n = 30.

This matters because workflow consolidation is often recommended as a CI efficiency measure in DevOps literature and tooling documentation. This study's data suggests that consolidation should be understood as an operational improvement (simpler maintenance, enforced fail-fast semantics, better readability) rather than an energy optimisation. Its value in this study is as a precondition for applying path filters uniformly (C4), not as a standalone intervention.

### C. Path Filtering and the Combined Strategy (RQ3)

The marginal energy difference between C2 and C4 in this study (583.2 J vs. 548.4 J; 6.0% additional reduction) reflects workflow consolidation's modest operational savings. Path filtering's energy benefit is not captured by this study's `workflow_dispatch` execution protocol, since `workflow_dispatch` events bypass path filters by GitHub Actions design. In real-world usage, however, path filtering is potentially the highest-leverage intervention of the three: a run that is never triggered saves 100% of that run's energy. For a project where developers regularly push documentation updates, tooling configuration changes, or non-code assets, the fraction of runs that path filtering would suppress can be substantial.

The C4 combination — consolidation + caching + path filtering — therefore represents the full stack of available optimisations: per-run energy is reduced by caching, per-run operational overhead is reduced by consolidation, and the total number of runs is reduced by path filtering. Only the first of these is captured in this study's controlled measurement.

### D. Regional Carbon Sensitivity and Infrastructure Implications (RQ3)

The 13.8× Norway–Ireland ratio in Table III represents carbon inequality at the infrastructure layer. Two identical C1 pipelines, one executing on a Norwegian-region runner and one on an Irish-region runner, produce 0.00588 gCO₂eq and 0.0812 gCO₂eq per run respectively — a difference that no workflow optimisation can close while remaining in Ireland. The Singapore runner produces 0.0960 gCO₂eq per C1 run, 16.3× higher than Norway.

For organisations operating at scale, runner location selection is therefore a high-leverage carbon reduction lever that requires no code changes and is immediately available through self-hosted runners in renewable-energy regions or enterprise GitHub runner configurations. This study's SCI analysis quantifies that lever for the first time in the context of a CI/CD pipeline.

---

## VIII. Threats to Validity

### A. Construct Validity

*Does the measurement instrument (Eco-CI joule values) actually capture what the study claims to measure (CI pipeline energy consumption)?*

Eco-CI estimates energy via a CPU utilisation model rather than direct hardware measurement. The model maps CPU utilisation to power draw using per-processor SPECpower data and integrates over elapsed time. For GitHub-hosted `ubuntu-latest` runners on Azure infrastructure (Intel Xeon Platinum series), the model is well-characterised. However, two limitations apply:

1. Eco-CI captures CPU energy only, not DRAM, network I/O, or storage energy. The dependency-installation stage involves substantial network I/O that contributes to total system power draw but is not captured by the CPU model. The measured reductions are therefore a lower bound on the actual energy reduction from caching.

2. Eco-CI does not measure GPU or memory bandwidth energy. For the CPU-bound workloads in this study (Python test execution, pip package resolution) this is not a concern, but it limits applicability of the methodology to GPU-intensive CI workloads without modification.

### B. Internal Validity

*Are confounds controlled? Are observed differences caused by the independent variable (configuration) rather than extraneous factors?*

GitHub-hosted runners operate in a shared multi-tenant environment where CPU, memory, and network resources fluctuate between runs for identical workloads. This is the primary source of run-to-run variability, visible in the standard deviations in Table IV (SD ≈ 11–12% of mean). Three controls mitigate this threat: (1) 30 repetitions per configuration, (2) non-parametric statistical tests that make no distributional assumptions, and (3) a minimum five-minute inter-run interval to reduce serial correlation between successive runs on shared infrastructure.

A potential confound is GitHub Actions caching behaviour on successive runs: the pip cache is populated on the first C2/C4 run and reused thereafter. The first run in each cached configuration will have higher energy (cache miss) than subsequent runs (cache hit). This effect is mitigated by including all 30 runs in the analysis and reporting medians alongside means; removing the first run from each cached configuration did not materially change the results.

### C. External Validity

*Do the findings generalise beyond the specific subject and platform studied?*

HTTPie CLI represents a specific project archetype: a production-grade Python library, medium-sized test suite (~90 seconds per matrix cell), stable dependency set. The absolute energy values (C1 mean = 847.3 J) will not transfer directly to projects with significantly different dependency graphs or test suite durations. However, the directional findings — that caching targets the dependency-installation stage specifically, and that consolidation alone is not an energy reduction — are mechanistically grounded and expected to generalise broadly across Python CI pipelines.

The study uses GitHub-hosted runners only. Self-hosted runners, GitLab CI, and other platforms have different runner lifecycle overhead, caching mechanisms, and scheduling behaviours, limiting direct transfer of findings to those environments.

### D. Conclusion Validity

*Are the statistical procedures appropriate for the data and the inferences drawn?*

The choice of Wilcoxon signed-rank test is justified by the non-normal distribution of the dependency-installation stage (Table VI). Applying a paired t-test to these distributions would inflate Type I error. The Bonferroni correction for three comparisons is conservative (it increases the risk of Type II error) but appropriate given that the three research questions are evaluated simultaneously on the same dataset. The n = 30 sample size provides greater than 95% power for detecting medium effect sizes (|δ| > 0.33) at α = 0.017; the large effects observed (|δ| = 0.89, 0.91 for C2 and C4) are well within the detectable range. The negligible Cliff's delta for C3 (|δ| = 0.14) confirms that the failure to reject the null hypothesis for consolidation is not purely a power limitation.

---

## IX. Conclusion

This paper presents an empirical methodology for measuring and optimising the carbon footprint of CI/CD pipelines, grounded in the SCI specification (ISO/IEC 21031:2024) and operationalised via the Eco-CI energy estimation tool on GitHub Actions. Four progressively optimised configurations of the HTTPie CLI pipeline are evaluated across 30 repeated controlled runs each, producing the first SCI-compliant empirical carbon measurement of a production open-source Python CI/CD pipeline.

Three findings have direct practical relevance:

1. **Pip dependency caching is the dominant energy lever.** Adding `cache: pip` to `actions/setup-python` is a one-line change that produces a statistically significant, large-effect reduction of 31.2% in per-run energy consumption (p < 0.001, Cliff's δ = −0.89) by eliminating redundant PyPI downloads on every run.

2. **Workflow consolidation alone is not an energy optimisation.** Merging three workflow files into one does not reach statistical significance at the Bonferroni-corrected threshold (p = 0.031 vs. α = 0.017, δ = −0.14 negligible). Its value is operational — simpler maintenance, fail-fast semantics — but should not be presented as an energy reduction in isolation.

3. **Runner location is a high-leverage carbon variable.** The Norway–Ireland grid intensity ratio (25:345, approximately 13.8×) produces a per-run carbon differential that is larger than all configuration-level optimisations combined. Organisations able to select lower-carbon runner regions can achieve greater carbon reductions through infrastructure choice than through any workflow change studied here.

The audit methodology is deliberately general: any team using GitHub Actions can reproduce the full experiment within a working day using the publicly available instrumented fork, collection scripts, and analysis notebook. As regulatory pressure from the CSRD and comparable frameworks intensifies and software carbon accounting tooling matures, empirical pipeline audits of this kind will become a routine element of responsible software engineering practice.

**Future work** includes extending the methodology to multi-language ecosystems, characterising the energy impact of test parallelism strategies and distributed test execution, evaluating self-hosted runners in renewable-energy regions, and developing automated CI carbon dashboards that surface SCI scores as continuous metrics alongside test pass rates and coverage percentages.

---

## Data Availability and Replication Package

All artefacts required to replicate this study are publicly available at:

**https://github.com/Umer-2612/httpie-cli-carbon-study**

The repository contains:
- Four instrumented experiment branches (`experiment/c1-baseline`, `experiment/c2-pip-cache`, `experiment/c3-consolidation`, `experiment/c4-combined`), each independently executable via `workflow_dispatch`
- `scripts/collect_results.py` — data collection script (GitHub Actions API)
- `analysis/energy_analysis.ipynb` — full statistical analysis notebook (Python, scipy, pandas, matplotlib)
- `results/raw_data.csv` — consolidated Eco-CI measurements (30 runs × 4 configurations × 6 stages)
- This paper in Markdown format

The repository is licensed under MIT. Instructions for reproducing the full experiment are provided in `README.md`.

---

## Acknowledgements

The author thanks the HTTPie project maintainers for producing and openly maintaining the software system used as the experimental subject, and the Green Coding Solutions team for developing and maintaining the Eco-CI tool.

---

## References

[1] International Energy Agency, *Data Centres and Data Transmission Networks*, IEA, Paris, 2023. [Online]. Available: https://www.iea.org/energy-system/buildings/data-centres-and-data-transmission-networks

[2] E. Masanet, A. Shehabi, N. Lei, S. Smith, and J. Koomey, "Recalibrating Global Data Center Energy-Use Estimates," *Science*, vol. 367, no. 6481, pp. 984–986, Feb. 2020, doi: 10.1126/science.aba3758.

[3] G. Pinto and F. Castor, "Energy Efficiency: A New Concern for Application Software Developers," *Communications of the ACM*, vol. 60, no. 12, pp. 68–75, Dec. 2017, doi: 10.1145/3154384.

[4] M. Hilton, T. Tunnell, K. Huang, D. Marinov, and D. Dig, "Usage, Costs, and Benefits of Continuous Integration in Open-Source Projects," in *Proc. 31st IEEE/ACM Int. Conf. Automated Software Engineering (ASE)*, Singapore, 2016, pp. 426–437, doi: 10.1145/2970276.2970358.

[5] Green Software Foundation, *Software Carbon Intensity (SCI) Specification*, v1.0, GSF, 2022. Adopted as ISO/IEC 21031:2024. [Online]. Available: https://sci-guide.greensoftware.foundation

[6] Green Coding Solutions, *Eco-CI Energy Estimation*, GitHub Repository, 2023. [Online]. Available: https://github.com/green-coding-solutions/eco-ci-energy-estimation

[7] R. Pereira, M. Couto, F. Ribeiro, R. Rua, J. Cunha, J. P. Fernandes, and J. Saraiva, "Energy Efficiency across Programming Languages: How Do Energy, Time, and Memory Relate?" in *Proc. 10th ACM SIGPLAN Int. Conf. Software Language Engineering (SLE)*, Vancouver, 2017, pp. 256–267, doi: 10.1145/3136014.3136031.

[8] HTTPie, *HTTPie CLI — A Modern, User-Friendly HTTP Client*, GitHub Repository, 2024. [Online]. Available: https://github.com/httpie/cli

[9] European Commission, "Directive 2022/2464 of the European Parliament and of the Council — Corporate Sustainability Reporting Directive (CSRD)," *Official Journal of the European Union*, L 322, pp. 15–80, Dec. 2022.

[10] GitHub Inc., *GitHub Actions Documentation — Events that Trigger Workflows*, 2024. [Online]. Available: https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows

[11] C. Wohlin, P. Runeson, M. Höst, M. C. Ohlsson, B. Regnell, and A. Wesslén, *Experimentation in Software Engineering*. Springer, Berlin, 2012, doi: 10.1007/978-3-642-29044-2.

[12] J. Romano, J. D. Kromrey, J. Coraggio, and J. Skowronek, "Appropriate Statistics for Ordinal Level Data: Should We Really Be Using t-Test and Cohen's d for Evaluating Group Differences on the NSSE and Other Surveys?" in *Florida Association of Institutional Research*, 2006.
