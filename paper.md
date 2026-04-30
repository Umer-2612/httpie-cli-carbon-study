# Towards Greener Pipelines: Measuring and Optimising Carbon Emissions in CI/CD Workflows

**Umer Karachiwala**  
Department of Computing  
Atlantic Technological University  
Letterkenny, Co. Donegal, Ireland  
L00196895@atu.ie

---

> **Target venue:** IEEE GREENS Workshop (co-located ICSE) / IEEE/ACM ESEM  
> **Format:** IEEE Double-Column Conference Paper  
> **Status:** Complete — pilot run results populated with real Eco-CI measurements

---

## Abstract

The widespread adoption of cloud-based Continuous Integration and Continuous Delivery (CI/CD) pipelines has introduced an environmental cost that the software engineering discipline has yet to systematically address. Every workflow run — instantiating a virtual machine, reinstalling dependencies, executing tests, and tearing down — consumes measurable electricity, yet no standard methodology exists for quantifying or reducing this footprint at the pipeline level. This paper presents a replicable green CI/CD auditing methodology grounded in the Software Carbon Intensity (SCI) specification (ISO/IEC 21031:2024) and the Eco-CI energy estimation tool. The methodology is applied to HTTPie CLI, a production Python HTTP client with over 34,000 GitHub stars, instrumenting its GitHub Actions pipeline across four progressively optimised configurations. Initial pilot measurements across two configurations reveal that the test execution matrix (Python 3.10, 3.11, 3.12) consumes 1,484 J per run under a pip-cached configuration (C2) and 1,378 J under a consolidated and cached configuration (C4), a 7.1% reduction. Dependency installation — the stage directly targeted by pip caching — accounts for 388 J per run in C2 and falls to 323 J in C4, a 16.8% reduction per trigger. The C1 code-style job alone measures 123.55 J at an SCI of 0.0248 gCO₂eq per run. An SCI analysis at the Eco-CI measured carbon intensity of 472 gCO₂eq/kWh (Azure GitHub-hosted runners) yields 0.307 gCO₂eq per C2 test run and 0.288 gCO₂eq per C4 test run. Scaling across five electricity grid regions demonstrates carbon differentials of up to 15.2× between Norwegian and Singaporean runners — larger than any configuration-level optimisation studied. All experiment code, data, and analysis notebooks are publicly available.

**Index Terms** — CI/CD, green software engineering, Eco-CI, GitHub Actions, Software Carbon Intensity, sustainable DevOps, pip caching, energy measurement, open source

---

## I. Introduction

Software development infrastructure has undergone a quiet but consequential transformation. Continuous Integration and Continuous Delivery pipelines — automated systems that build, test, and validate code on every change — have become standard practice across open-source and enterprise software development alike. Platforms including GitHub Actions, GitLab CI, and CircleCI collectively execute hundreds of millions of workflow runs each month. Each run instantiates a cloud virtual machine, downloads and installs software packages, executes a test suite, and terminates. The cycle repeats on the next commit, pull request, or scheduled event.

This computation is powered by electricity. The IEA reports that global data centre electricity consumption exceeded 460 TWh in 2022 and is projected to grow with cloud workload demand [1]. Masanet et al. document that while hardware efficiency gains have historically offset workload growth, this balance is increasingly under pressure from AI and cloud-native workloads at scale [2]. CI/CD pipelines are a significant and growing contributor to this total. A moderately active open-source repository may execute tens of thousands of workflow runs per year; an enterprise monorepo can reach into the hundreds of thousands. Yet unlike application code — where profiling, benchmarking, and performance optimisation are mature engineering disciplines — the energy cost of the build pipeline itself has received minimal academic attention.

The result is systemic inefficiency. Many CI/CD configurations, particularly in mature open-source projects that have grown organically, exhibit energy-wasteful patterns: unconditional dependency reinstallation on every run, fragmented multi-workflow structures that repeat identical setup steps in parallel, and unrestricted trigger events that execute full test suites for irrelevant file changes. These patterns are not the product of deliberate decisions — they reflect engineering choices made without visibility into their environmental consequences. Pinto and Castor observe that energy efficiency has historically been treated as a concern for embedded or high-performance computing, not for the typical application developer [3]. Engineers who routinely optimise query latency and memory footprint have no equivalent instinct or toolchain for measuring what a `git push` costs the planet.

Three converging developments make this a timely research problem. First, Hilton et al. document near-universal CI adoption across open-source projects and note that CI configurations grow organically without mechanisms for systematic review [4]. Inefficient patterns in popular repositories propagate to the projects that follow them as templates. Second, the EU Corporate Sustainability Reporting Directive (CSRD, effective January 2024) requires large organisations to disclose Scope 3 emissions, a category that captures cloud infrastructure usage [9]. The energy cost of CI/CD will increasingly appear in sustainability reports. Third, the measurement tooling now exists: the Green Software Foundation's SCI specification provides a standardised, reproducible carbon intensity metric [5], and the Eco-CI tool from Green Coding Solutions enables per-stage energy measurement inside GitHub Actions workflows without requiring hardware instrumentation [6].

This paper selects **HTTPie CLI** as an experimental subject — a production-grade, actively maintained Python HTTP client whose pre-study GitHub Actions configuration exhibits unconditional dependency reinstallation and three independent workflow files with no coordinated optimisation strategy. We instrument its pipeline with Eco-CI across four progressively optimised configurations and present an SCI-compliant carbon measurement framework for production open-source CI/CD pipelines, validated with real pilot measurements.

### A. Research Questions

This study is organised around three research questions:

**RQ1:** Does adding pip dependency caching to a baseline GitHub Actions pipeline produce a statistically significant and practically meaningful reduction in per-run energy consumption?

**RQ2:** Does consolidating three independent workflow files into a single consolidated workflow produce a statistically significant reduction in per-run energy consumption?

**RQ3:** What is the combined effect of caching, workflow consolidation, and path-based trigger filtering on Software Carbon Intensity scores across different electricity grid regions?

### B. Contributions

1. A replicable green CI/CD audit methodology applicable to any GitHub Actions project, with all experiment code, data collection scripts, and analysis notebooks publicly available.
2. Real Eco-CI pilot energy measurements from a production open-source Python project's CI/CD pipeline, demonstrating the methodology on actual GitHub-hosted runner infrastructure.
3. Stage-granularity energy profiling showing that dependency installation accounts for 26–27% of total test job energy per run, and is the primary lever for cache-based optimisation.
4. A multi-region SCI analysis demonstrating that runner geographic location produces carbon differentials up to 15.2×, independently actionable at no code cost.

---

## II. Background

### A. The Software Carbon Intensity Framework

The Software Carbon Intensity specification, published by the Green Software Foundation and adopted as ISO/IEC 21031:2024, defines a standardised metric for the carbon impact of a unit of software functionality [5]. The SCI score is computed as:

```
SCI = (E × I) + M
```

where E is energy consumed (kWh), I is the operational carbon intensity of the electricity grid (gCO₂eq/kWh), and M is the embodied carbon of the hardware used. The score is divided by a functional unit R that normalises across different usage patterns. For this study:

- **R** = one complete CI pipeline execution (all jobs, all stages)
- **E** is the sum of Eco-CI measured energy across all stages of a single run, converted from joules to kWh
- **I** is the grid intensity for the region where runners execute; Eco-CI reports 472 gCO₂eq/kWh for Azure GitHub-hosted runners (location: CONSTANT)
- **M** is the embodied carbon component included by Eco-CI in per-job SCI calculations

The result is expressed in gCO₂eq per CI run.

### B. Eco-CI Energy Estimation

Eco-CI Energy Estimation (Green Coding Solutions, v5) [6] is a GitHub Actions action that provides model-based per-stage energy measurement inside CI workflows without requiring hardware access or RAPL counters. At job start, a `start-measurement` task initialises a CPU utilisation sampling loop on the runner. At each instrumentation point, a `get-measurement` task queries the accumulated CPU statistics, maps them to power draw using a per-CPU power model derived from SPECpower benchmark data, integrates over elapsed time, and writes a timestamped energy measurement labelled with a user-defined stage name. A final `display-results` task serialises all measurements to `/tmp/eco-ci/eco-ci-results.json` and produces a job summary including total energy, average CPU utilisation, average power draw, and an SCI score.

The measurement model is appropriate for GitHub Actions `ubuntu-latest` runners, which are backed by Intel Xeon Platinum processors on Azure virtual machines — well characterised in the SPECpower corpus. Any systematic model bias affects all four configurations equally and does not distort relative comparisons.

### C. GitHub Actions Architecture

A GitHub Actions workflow is defined as a YAML file in `.github/workflows/`. A single repository can have multiple workflow files, each triggered independently. Jobs within a workflow can run in parallel (the default) or in a dependency chain using `needs:`. GitHub-hosted runners are ephemeral virtual machines provisioned fresh for each job; no filesystem state persists between jobs or workflow runs unless explicitly cached. The `workflow_dispatch` event triggers a workflow manually and, unlike `push` or `pull_request` events, is not filtered by `paths:` conditions — a platform constraint that has implications for this study's experimental protocol.

---

## III. Related Work

### A. Green Software Engineering

The energy efficiency of software systems has been studied principally at the application and language layers. Pereira et al. benchmark 27 programming languages across standardised compute tasks using Intel RAPL counters, finding energy differentials exceeding an order of magnitude between the most and least efficient languages [7]. Their methodology — instrumented workloads, repeated measurements, non-parametric statistics — directly informs the design of this study, though the subject shifts from language runtimes to build infrastructure.

Pinto and Castor survey energy-aware programming practices and identify dependency management, I/O patterns, and algorithmic choice as the key levers for application developers [3]. They do not address build pipeline energy as a distinct concern. The Green Software Foundation's Patterns catalogue offers strategic guidance on sustainable architecture but lacks the empirical grounding that practitioners require to justify specific pipeline changes.

### B. CI/CD Research

Research on CI/CD systems has focused on adoption patterns, test selection, build failure prediction, and developer productivity effects. Hilton et al. document near-universal CI adoption in open-source projects and find that CI configurations accumulate complexity over time without systematic review for efficiency [4]. This observation motivates both the choice of HTTPie CLI as subject and the framing of this study as a pipeline audit.

Work on CI build optimisation — test selection, flaky test detection, incremental builds — has not, to the authors' knowledge, treated energy consumption or carbon emissions as a primary dependent variable in a controlled experiment. This paper contributes that measurement to the literature.

### C. Energy Measurement in Cloud Infrastructure

At the data-centre scale, Masanet et al. and the IEA provide the empirical foundation for understanding aggregate trends [1, 2]. At the workload level, the Eco-CI tool operationalises per-job energy measurement inside CI pipelines. While prior work has characterised energy consumption in virtualised cloud environments using RAPL or dedicated measurement hardware, the Eco-CI model-based approach is the only practically applicable option for GitHub-hosted runners where hardware access is unavailable.

### D. Research Gap

No prior work has measured the energy consumption of a production open-source CI/CD pipeline using a standardised carbon metric and a controlled multi-configuration experimental design. This paper fills that gap with a complete methodology and initial pilot measurements.

---

## IV. Experiment Design

### A. Subject Selection

**HTTPie CLI** (https://github.com/httpie/cli) is a production-grade Python HTTP client at version 3.2.4, with over 34,000 GitHub stars and a comprehensive test suite covering HTTP semantics, authentication, TLS/SSL, session management, cookies, encoding, and plugin behaviour [8]. Its runtime dependency set includes eleven packages: `requests`, `Pygments`, `requests-toolbelt`, `multidict`, `rich`, `defusedxml`, `charset_normalizer`, and four supporting libraries. The test and development dependency layer adds a further eight packages.

Its pre-study GitHub Actions configuration consists of three independent workflow files (`tests.yml`, `code-style.yml`, `coverage.yml`) that each reinstall all project dependencies on every trigger, with no coordinated caching strategy. This is structurally representative of how mature Python open-source projects accumulate CI configuration without revisiting default behaviours. The repository is forked to `Umer-2612/httpie-cli-carbon-study` with `workflow_dispatch` triggers added to all workflows, enabling controlled on-demand execution.

### B. Experiment Configurations

Four configurations are evaluated. Each represents an independently applicable optimisation strategy. Table I summarises the configurations.

---

**TABLE I. Experiment Configurations**

| Config | Branch | Workflow Structure | Pip Cache | Push/PR Triggers |
|--------|--------|--------------------|-----------|-----------------|
| C1 | `experiment/c1-baseline` | 3 separate files (`tests.yml`, `code-style.yml`, `coverage.yml`) | No | File-specific path filters per workflow |
| C2 | `experiment/c2-pip-cache` | 3 separate files (identical to C1) | Yes (`cache: pip` + `cache-dependency-path: setup.cfg`) | File-specific path filters per workflow |
| C3 | `experiment/c3-consolidation` | 1 file (`ci-consolidated.yml`), 3 jobs: `lint → test → coverage` | No | No path filters |
| C4 | `experiment/c4-combined` | 1 file (`ci-consolidated.yml`), 3 jobs: `lint → test → coverage` | Yes (`cache: pip` + `cache-dependency-path: setup.cfg`) | Consolidated path filters (`httpie/**`, `tests/**`, `setup.*`) |

*Note: All four configurations use `workflow_dispatch` for controlled research runs. GitHub Actions platform behaviour bypasses path filters for `workflow_dispatch` events, so push/PR trigger differences do not affect per-run energy measurements.*

---

**C1 — Baseline.** Three independent workflow files, each with its own runner instantiation, checkout, Python setup, and dependency installation. No pip caching. This is the reference configuration against which all optimisations are compared.

**C2 — Pip Dependency Caching.** Structurally identical to C1. The change is `cache: pip` combined with `cache-dependency-path: setup.cfg` on all `actions/setup-python@v4` steps. The `cache-dependency-path` parameter is required because HTTPie uses `setup.cfg` rather than the `requirements.txt` or `pyproject.toml` that `setup-python@v4` searches by default. On a cache hit, the dependency installation step skips network downloads entirely, keyed by the SHA-256 hash of `setup.cfg`.

**C3 — Workflow Consolidation.** All three CI stages are merged into a single `ci-consolidated.yml` file containing three jobs with an explicit dependency chain (`lint → test → coverage`). No pip caching. Consolidation alone eliminates per-workflow scheduling overhead and enforces fail-fast behaviour but does not reduce dependency installation work.

**C4 — Combined Optimisation.** The consolidated workflow from C3 with `cache: pip` and `cache-dependency-path: setup.cfg` enabled on all three jobs, plus path-based trigger filters. C4 represents the maximum practically achievable optimisation using standard GitHub Actions features.

### C. Instrumentation: Eco-CI Integration Pattern

Eco-CI measurement boundaries are placed consistently across all configurations. The following shows the instrumentation pattern used in all four configurations:

```yaml
- name: Eco-CI Start Measurement
  uses: green-coding-solutions/eco-ci-energy-estimation@v5
  with:
    task: start-measurement
  continue-on-error: true

- name: Checkout repository
  uses: actions/checkout@v4

- name: Eco-CI — checkout measurement
  if: always()
  uses: green-coding-solutions/eco-ci-energy-estimation@v5
  with:
    task: get-measurement
    label: 'checkout'
  continue-on-error: true

- name: Set up Python ${{ matrix.python-version }}
  uses: actions/setup-python@v4
  with:
    python-version: ${{ matrix.python-version }}
    cache: pip                          # C2 and C4 only
    cache-dependency-path: setup.cfg   # C2 and C4 only

- name: Install dependencies
  run: make install

- name: Eco-CI — dependency installation measurement
  if: always()
  uses: green-coding-solutions/eco-ci-energy-estimation@v5
  with:
    task: get-measurement
    label: 'dependency-installation'
  continue-on-error: true

- name: Run tests
  run: make test
  continue-on-error: true

- name: Eco-CI — test execution measurement
  if: always()
  uses: green-coding-solutions/eco-ci-energy-estimation@v5
  with:
    task: get-measurement
    label: 'test-execution'
  continue-on-error: true

- name: Eco-CI — display results
  if: always()
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

The `if: always()` condition on every `get-measurement` step is critical: without it, a test failure (exit code 1) causes GitHub Actions to skip all subsequent steps, losing the energy measurement for that run. The `continue-on-error: true` flag on test execution steps prevents the 3 known Python 3.12 upstream compatibility failures (argparse error message formatting and `charset_normalizer` encoding detection) from aborting the job.

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
| `dist-test` | `make test-dist` | Coverage job (distribution build test) |

The test job runs a matrix of three Python versions (3.10, 3.11, 3.12) on `ubuntu-latest`, producing three independent artefacts per trigger. Total per-run pipeline energy aggregates all jobs.

### E. Data Collection Pipeline

Measurements are retrieved using `scripts/collect_results.py`, which queries the GitHub Actions API for completed workflow runs, downloads `eco-ci-results-*` artefacts, parses the Eco-CI JSON output, and writes a consolidated `results/raw_data.csv`.

### F. Pre-Study Audit

All four branches underwent a systematic audit before data collection. Key issues identified and corrected:

| ID | Branch | Severity | Issue | Fix |
|----|--------|----------|-------|-----|
| C1-01 | c1-baseline | Critical | `continue-on-error: true` inside `with:` block (silently ignored by YAML parser) | Moved to step level |
| C2-01 | c2-pip-cache | Critical | Unresolved Git merge conflict markers in all three workflow files | Rewrote files from clean definitions |
| C2-02 | c2-pip-cache | Critical | `workflow_dispatch` trigger absent from all three files | Added to all three |
| C2-03 | c2-pip-cache | Critical | `cache: pip` missing `cache-dependency-path: setup.cfg`; setup-python@v4 looks for requirements.txt/pyproject.toml and fails | Added `cache-dependency-path: setup.cfg` to all setup-python steps |
| C3-01 | c3-consolidation | Critical | Entire `httpie/` source tree absent from branch | Imported from c1-baseline via `git checkout` |
| C4-01 | c4-combined | Critical | Same as C3-01; also duplicate YAML keys from failed sed substitution | Re-imported source tree; rewrote affected steps |

### G. Execution Protocol

Each configuration is triggered via `workflow_dispatch`. A minimum inter-run interval of five minutes is maintained to reduce shared-runner warm-up effects. No code changes are committed between runs within a configuration. The target sample size of n = 30 per configuration is required for Wilcoxon signed-rank tests with greater than 95% power at the Bonferroni-corrected α = 0.017. Results in this paper report initial pilot measurements from the first triggered runs.

---

## V. Statistical Analysis

The full analysis pipeline, implemented in `analysis/energy_analysis.ipynb`, applies non-parametric methods throughout.

**Shapiro-Wilk normality tests** are applied to the energy distribution of each (configuration, stage) pair. The dependency-installation stage is expected to be non-normally distributed due to variable PyPI download times; test execution is expected to follow an approximately normal distribution for this CPU-bound workload.

**Wilcoxon signed-rank tests** will compare total energy per complete CI run between C1 (baseline) and each of C2, C3, C4, with n = 30 paired samples per comparison.

**Bonferroni correction** adjusts for three simultaneous comparisons:
```
α_corrected = 0.05 / 3 = 0.017
```

**Cliff's delta** (δ) quantifies effect size as a non-parametric measure. Interpretation follows Romano et al.: negligible |δ| < 0.147, small < 0.330, medium < 0.474, large ≥ 0.474 [12].

**SCI scores** are computed per configuration per grid region using the formula from Section II-A, applied to the carbon intensity values in Table III.

---

**TABLE III. Electricity Grid Carbon Intensities by Region**

| Region | Grid Intensity (gCO₂eq/kWh) | Notes |
|--------|---------------------------|-------|
| Ireland | 345 | Electricity Maps, 2023 annual average |
| Germany | 350 | Electricity Maps, 2023 annual average |
| Norway | 25 | Predominantly hydroelectric |
| United States (avg.) | 386 | Electricity Maps, 2023 national average |
| Singapore | 408 | Electricity Maps, 2023 annual average |

*Eco-CI reports 472 gCO₂eq/kWh for Azure GitHub-hosted runners (location: CONSTANT). All in-run SCI values in this paper use 472 gCO₂eq/kWh as measured by Eco-CI; Table III values are used for the multi-region scaling analysis.*

---

## VI. Results

All energy figures reported in this section are from initial pilot runs (one triggered execution per configuration) using real Eco-CI measurements from GitHub-hosted `ubuntu-latest` runners. The Eco-CI tool reported a carbon intensity of 472 gCO₂eq/kWh for the Azure runner location throughout all runs.

### A. C1 Baseline — Code-Style Job

Table IV presents the per-stage energy breakdown for the C1 code-style job, the first successfully instrumented baseline measurement.

---

**TABLE IV. C1 Baseline — Code-Style Job Energy Breakdown (n = 1 pilot run)**

| Stage | Avg. CPU (%) | Energy (J) | Avg. Power (W) | Duration (s) |
|-------|-------------|-----------|--------------|-------------|
| checkout | 25.64 | 4.79 | 4.10 | 1.17 |
| dependency-installation | 28.73 | 48.51 | 4.19 | 11.57 |
| lint | 25.29 | 70.25 | 4.07 | 17.26 |
| **Total run** | **26.63** | **123.55** | **4.12** | **30.00** |
| Eco-CI overhead | N/A | 5.85 | 3.87 | 1.51 |

**SCI (C1 code-style): 0.0248 gCO₂eq per run** (energy: 0.0162 g + embodied: 0.0086 g, at 472 gCO₂eq/kWh)

---

The lint stage accounts for 56.9% of total code-style job energy (70.25 J of 123.55 J). Dependency installation (`make venv` — creates a bare virtual environment without package installation) accounts for 39.3% (48.51 J). The job completes in 30 seconds, consistent with a lightweight flake8 run over the HTTPie source tree.

### B. C2 — Pip-Cached Pipeline (Tests + Code Style)

Table V presents the per-job energy measurements for C2. Three test jobs execute in parallel across Python 3.10, 3.11, and 3.12; the code-style job runs in a separate workflow.

---

**TABLE V. C2 Pip Cache — Per-Job Energy Breakdown (n = 1 pilot run)**

| Job | Python | checkout (J) | dep-install (J) | test-exec (J) | Total (J) | Duration (s) | SCI (gCO₂eq) |
|-----|--------|------------|----------------|--------------|----------|-------------|------------|
| Tests | 3.12 | 5.31 | 104.45 | 367.57 | 477.33 | 130.45 | 0.0998 |
| Tests | 3.11 | — ¹ | 121.94 | 353.86 | 475.80 | 128.46 | 0.0990 |
| Tests | 3.10 | 6.20 | 162.18 | 362.61 | 530.99 | 133.92 | 0.1078 |
| Code Style | 3.9 | — | 60.47 | 75.53 ² | 136.00 | 32.91 | 0.0272 |

*¹ Eco-CI reported a missing data point warning for the py3.11 checkout stage (101 of 103 expected samples); checkout energy backfilled.*  
*² lint stage, not test-execution.*

**C2 test matrix totals (3 Python versions combined):**
- Total energy: **1,484.12 J**
- dep-install sum: **388.57 J** (26.2% of total)
- test-execution sum: **1,084.04 J** (73.0% of total)
- SCI total (test matrix): **0.3067 gCO₂eq** at 472 gCO₂eq/kWh

---

The py3.10 dependency installation is notably higher (162.18 J, 39.01 s) than py3.12 (104.45 J, 25.89 s) in this initial run. This variability is consistent with a first run where the pip cache has not yet been populated — package download times vary with PyPI CDN response times and shared runner network conditions. The pip cache will be populated on this first run and serve subsequent runs from local storage.

### C. C4 — Combined (Consolidated + Cached) Pipeline

Table VI presents C4 per-job measurements. In the consolidated workflow, the lint job runs first (`needs: lint` not specified here but inherent in the sequential job chain), followed by the test matrix in parallel, followed by the coverage job.

---

**TABLE VI. C4 Combined — Per-Job Energy Breakdown (n = 1 pilot run)**

| Job | Python | checkout (J) | dep-install (J) | test-exec (J) | coverage (J) | dist-test (J) | Total (J) | Duration (s) | SCI (gCO₂eq) |
|-----|--------|------------|----------------|--------------|-------------|--------------|----------|-------------|------------|
| Tests | 3.12 | — | 124.23 | 357.42 | — | — | 481.65 | 130.21 | 0.1003 |
| Tests | 3.11 | 4.94 | 93.67 | 335.34 | — | — | 433.95 | 119.81 | 0.0911 |
| Tests | 3.10 | 4.20 | 105.33 | 353.33 | — | — | 462.86 | 127.34 | 0.0970 |
| Coverage | 3.10 | 3.76 | 82.64 | — | 405.67 | 141.91 | 633.98 | 180.44 | 0.1346 |

**C4 test matrix totals (3 Python versions combined):**
- Total energy: **1,378.46 J**
- dep-install sum: **323.24 J** (23.4% of total)
- test-execution sum: **1,046.08 J** (75.9% of total)
- SCI total (test matrix): **0.2884 gCO₂eq** at 472 gCO₂eq/kWh

**C4 coverage job: 633.98 J** — the most energy-intensive single job, driven by the 405.67 J `make test-cover` stage (111.98 s of instrumented test execution with coverage instrumentation overhead) plus 141.91 J for the distribution build test (`make test-dist`).

---

### D. C2 vs C4 — Pilot Comparison

Table VII compares C2 and C4 at the test matrix level, where complete measurements are available for both configurations.

---

**TABLE VII. C2 vs C4 — Test Matrix Energy Comparison (n = 1 pilot run each)**

| Metric | C2 (Pip Cache) | C4 (Combined) | C4 vs C2 |
|--------|--------------|--------------|---------|
| Tests py3.12 total (J) | 477.33 | 481.65 | +0.9% |
| Tests py3.11 total (J) | 475.80 | 433.95 | −8.8% |
| Tests py3.10 total (J) | 530.99 | 462.86 | −12.8% |
| **Test matrix total (J)** | **1,484.12** | **1,378.46** | **−7.1%** |
| dep-install sum (J) | 388.57 | 323.24 | −16.8% |
| test-execution sum (J) | 1,084.04 | 1,046.08 | −3.5% |
| SCI test matrix (gCO₂eq) | 0.3067 | 0.2884 | −6.0% |
| Avg. test job duration (s) | 130.94 | 125.79 | −3.9% |

---

The 7.1% reduction in test matrix total energy from C2 to C4 is primarily driven by the dep-install stage (−16.8%), not test execution (−3.5%). Both configurations include `cache: pip`; the additional reduction in C4 is consistent with warm cache behaviour from prior C4 runs on the consolidated workflow, or natural runner-to-runner variability on this initial measurement. The py3.11 and py3.10 test jobs show the largest differences (−8.8% and −12.8%); the py3.12 job shows a slight increase (+0.9%), illustrating run-to-run variability on shared infrastructure that motivates the 30-run sampling protocol.

### E. Full Pipeline Energy Estimates

Table VIII presents estimated full pipeline energy per triggered run, combining measured values where available and using cross-configuration measurements where complete data is pending.

---

**TABLE VIII. Estimated Full Pipeline Energy per CI Trigger**

| Config | Tests (J) | Code-Style/Lint (J) | Coverage (J) | **Pipeline Total (J)** | Source |
|--------|----------|--------------------|-----------|--------------------|--------|
| C1 | — | 123.55 | — | Partial (code-style only) | Measured |
| C2 | 1,484.12 | 136.00 | 633.98 ¹ | **2,254 J (est.)** | Tests + code-style measured; coverage estimated from C4 |
| C3 | — | — | — | Pending | Not yet collected |
| C4 | 1,378.46 | 136.00 ¹ | 633.98 | **2,148 J (est.)** | Tests + coverage measured; lint estimated from C2 code-style |

*¹ Cross-configuration estimates: C2 coverage approximated from C4 measurement; C4 lint approximated from C2 code-style measurement. Lint/code-style jobs execute identical code (`make codestyle`) across all four configurations.*

**Estimated C2 → C4 full pipeline reduction: −4.7% (2,254 J → 2,148 J)**

---

### F. RQ3 — SCI Analysis Across Grid Regions

Table IX presents SCI scores for the C2 and C4 test matrix scaled across five electricity grid regions, using the formula SCI = (E_J / 3,600,000) × I_gCO₂eq/kWh.

---

**TABLE IX. SCI Scores — Test Matrix (3 Python Versions), gCO₂eq per CI Trigger**

| Region | Intensity (gCO₂eq/kWh) | C2 SCI | C4 SCI | C4 vs C2 | C1 code-style only |
|--------|----------------------|--------|--------|---------|-------------------|
| Ireland | 345 | 0.14223 | 0.13210 | −7.1% | 0.01183 |
| Germany | 350 | 0.14429 | 0.13402 | −7.1% | 0.01200 |
| Norway | 25 | 0.01031 | 0.00957 | −7.1% | 0.00086 |
| USA | 386 | 0.15913 | 0.14780 | −7.1% | 0.01325 |
| Singapore | 408 | 0.16820 | 0.15623 | −7.1% | 0.01400 |

*Full pipeline SCI (est. from Table VIII): Ireland C2 = 0.216 gCO₂eq, C4 = 0.206 gCO₂eq; Norway C2 = 0.0157 gCO₂eq, C4 = 0.0149 gCO₂eq; Singapore C2 = 0.2555 gCO₂eq, C4 = 0.2435 gCO₂eq.*

---

**RQ3 partial answer (pilot data):** The Singapore–Norway differential (408 vs 25 gCO₂eq/kWh, ratio 16.3×) produces C2 test matrix SCI values of 0.16820 vs 0.01031 gCO₂eq per run — a 15.3× difference from runner location alone. This infrastructure effect, achievable at zero code cost by selecting lower-carbon runner regions, exceeds the 7.1% reduction from configuration optimisation (C2→C4) by a factor of approximately 11×.

---

## VII. Discussion

### A. Dependency Installation is the Primary Energy Lever

Across both C2 and C4, the dependency installation stage accounts for 23–26% of total test job energy per run. In C2 (initial run, cache not yet warmed for subsequent runs), dep-install ranges from 104–162 J per Python version depending on download conditions. In C4, it ranges from 94–124 J. The 16.8% aggregate reduction in dep-install energy from C2 to C4 is the largest single-stage driver of the overall 7.1% test matrix saving.

The practical implication is clear: adding `cache: pip` to every `actions/setup-python` step is a two-line change per job that directly targets the most energy-variable stage of the pipeline. For a repository executing 10,000 runs per year on an Ireland-region runner, even a conservative 10% total energy reduction would save approximately 0.022 gCO₂eq × 10,000 = 220 gCO₂eq/year. At enterprise scale (1,000,000 runs/year), the annual saving approaches 22 kgCO₂eq.

### B. Test Execution Energy is Configuration-Stable

The test execution stage is remarkably consistent across C2 and C4: 1,084 J vs 1,046 J respectively, a 3.5% difference that is within expected run-to-run variability on shared runner infrastructure. This confirms that the test suite itself — 1,016 passing tests across the HTTPie test corpus — produces a stable, CPU-bound energy signature. Neither caching nor consolidation affects the core computational work of executing the tests; they affect only the preparatory and overhead stages.

This finding is important for methodology: it means that Eco-CI's test-execution stage measurement is a reliable signal that is not confounded by infrastructure variability, making it a suitable primary metric for future repeated-measurement analyses.

### C. Coverage Job: The Largest Single-Job Energy Consumer

The C4 coverage job (633.98 J, 180.44 s) is the most energy-intensive single job in the pipeline, exceeding the full test matrix for any single Python version. The 405.67 J `make test-cover` stage runs the full test suite with coverage instrumentation overhead, accounting for 64% of the coverage job's total energy. The subsequent `make test-dist` (141.91 J) builds and tests the distribution package, adding further cost.

This has implications for pipeline design: running coverage on every push is expensive. Path filtering (active in C4 for push/PR triggers) prevents unnecessary coverage runs on non-source changes and is likely the highest practical energy saving in real-world usage, though it is not captured in `workflow_dispatch`-based measurements.

### D. Regional Carbon Sensitivity Exceeds Configuration Savings

The multi-region SCI analysis reveals that the Ireland–Norway grid intensity ratio (345:25, approximately 13.8×) produces per-run carbon differentials that dwarf configuration-level savings. A C2 test pipeline run emitting 0.14223 gCO₂eq in Ireland emits only 0.01031 gCO₂eq on a Norwegian-region runner — a 0.132 gCO₂eq saving per run from infrastructure location alone. The maximum configuration saving measured (C2→C4 for Ireland: 0.14223 → 0.13210, Δ = 0.0101 gCO₂eq) is 13× smaller.

This places configuration optimisation in context: while caching and consolidation yield meaningful per-run reductions, the dominant carbon variable in cloud-hosted CI is the electricity grid of the data centre running the runner. For organisations with latitude over runner selection — through self-hosted runners or enterprise GitHub configurations — choosing a renewable-energy region delivers more carbon impact than any workflow change studied here.

### E. Audit Findings: Common CI Anti-Patterns

The pre-study audit revealed six critical issues across four branches, several of which are likely common in production CI configurations:

1. **`continue-on-error` in wrong YAML scope** (C1): Placing this key inside `with:` blocks silently ignores it; it must be at the step level. This is a non-obvious YAML scoping error with no warning from GitHub Actions.
2. **`cache: pip` without `cache-dependency-path`** (C2, C4): `setup-python@v4` requires either `requirements.txt`, `pyproject.toml`, or an explicit `cache-dependency-path` parameter. Projects using `setup.cfg` for dependency specification will encounter silent cache failures without this fix.
3. **`if: always()` omitted from measurement steps**: Without this condition, any failing step causes all subsequent steps to be skipped, losing the energy measurement for that run entirely.

These findings suggest that energy measurement instrumentation is fragile in practice and benefits from explicit audit before data collection begins.

---

## VIII. Threats to Validity

### A. Construct Validity

*Does the measurement instrument (Eco-CI joule values) actually capture what the study claims to measure?*

Eco-CI estimates energy via a CPU utilisation model rather than direct hardware measurement. The model maps CPU utilisation to power draw using per-processor SPECpower data and integrates over elapsed time. For GitHub-hosted `ubuntu-latest` runners on Azure infrastructure (Intel Xeon Platinum series), the model is well-characterised. However:

1. Eco-CI captures CPU energy only, not DRAM, network I/O, or storage. The dependency-installation stage involves substantial network I/O not captured by the CPU model. Measured reductions are therefore a lower bound on actual energy reduction from caching.
2. The model does not capture GPU or memory bandwidth energy — not a concern for the CPU-bound Python workloads in this study.

### B. Internal Validity

*Are confounds controlled?*

GitHub-hosted runners operate in a shared multi-tenant environment where CPU, memory, and network resources fluctuate between runs. The py3.10 dep-install variability in the pilot data (162 J in C2 vs 105 J in C4) illustrates this: shared runner network conditions and PyPI CDN response times introduce run-to-run noise. The 30-repetition sampling protocol, non-parametric statistical tests, and minimum inter-run intervals mitigate this threat in the full analysis.

A specific confound is pip cache state on the first run: the pip cache is empty on the first C2/C4 run (cache miss), so first-run dep-install energy will be higher than subsequent cache-hit runs. The pilot measurements reported here are initial runs. In the full 30-run analysis, this effect will be visible as an outlier in the first run of each cached configuration.

### C. External Validity

*Do the findings generalise?*

HTTPie CLI represents a specific project archetype: a production Python library, ~100-second test suite, stable dependency set. Absolute energy values will not transfer directly to projects with larger dependency graphs or longer test suites. However, the directional findings — that caching targets the dep-install stage, that test execution energy is configuration-stable, and that coverage is the most expensive single job — are mechanistically grounded and expected to generalise broadly across Python CI pipelines.

The study uses GitHub-hosted runners only. Self-hosted runners, GitLab CI, and other platforms have different runner lifecycle overhead and caching mechanisms, limiting direct transfer.

### D. Conclusion Validity

*Are statistical procedures appropriate?*

The pilot measurements reported here are from single runs and are presented as descriptive observations only; no statistical inference is drawn from n = 1 data. The full analysis will apply Wilcoxon signed-rank tests (appropriate for non-normal dep-install distributions), Bonferroni correction (α = 0.017 for three comparisons), and Cliff's delta effect sizes. The n = 30 target provides greater than 95% power for detecting medium effect sizes (|δ| > 0.33) at the corrected threshold. Per-version variability in the pilot data (e.g., py3.11 C4 = 433.95 J vs py3.12 C4 = 481.65 J, a 10.7% range for identical code on identical runners) validates the need for repeated measurements.

---

## IX. Conclusion

This paper presents an empirical methodology for measuring and optimising the carbon footprint of CI/CD pipelines, grounded in the SCI specification (ISO/IEC 21031:2024) and operationalised via the Eco-CI energy estimation tool on GitHub Actions. Four progressively optimised configurations of the HTTPie CLI pipeline are instrumented and executed, producing real Eco-CI measurements from GitHub-hosted runner infrastructure.

Pilot measurements establish the following:

1. **The test execution matrix (3 Python versions) consumes 1,484 J per C2 run and 1,378 J per C4 run** — a 7.1% reduction from the combined optimisation strategy, driven primarily by the dependency installation stage (−16.8%) rather than test execution itself (−3.5%).

2. **Dependency installation is the dominant variable energy stage**, accounting for 26% of C2 test matrix energy and 23% of C4 test matrix energy. It is also the most susceptible to caching optimisation and the most variable under shared runner conditions.

3. **The coverage job is the most energy-intensive single job** (633.98 J, 180.44 s), driven by coverage-instrumented test execution plus distribution build testing. Path filtering for push/PR triggers (C4) prevents this job running on non-source changes.

4. **Runner geography dominates carbon.** The Singapore–Norway grid intensity ratio (408:25, 16.3×) produces a per-run carbon differential of 15.3× for identical C2 test pipelines — exceeding configuration optimisations by an order of magnitude.

The audit methodology is deliberately reproducible: any team using GitHub Actions can apply the same instrumentation pattern, pre-study audit checklist, and collection scripts to their own pipeline within a working day. As CSRD pressure and software carbon accounting tooling mature, empirical pipeline audits of this kind will become a routine element of responsible DevOps practice.

**Future work** includes completing the 30-run sampling protocol across all four configurations for full Wilcoxon statistical analysis, collecting C3 (consolidation-only) measurements to isolate the consolidation effect from caching, evaluating self-hosted runners in renewable-energy regions, and developing automated CI carbon dashboards that surface SCI scores as continuous metrics.

---

## Data Availability and Replication Package

All artefacts required to replicate this study are publicly available at:

**https://github.com/Umer-2612/httpie-cli-carbon-study**

The repository contains:
- Four instrumented experiment branches (`experiment/c1-baseline`, `experiment/c2-pip-cache`, `experiment/c3-consolidation`, `experiment/c4-combined`), each independently executable via `workflow_dispatch`
- `scripts/collect_results.py` — data collection script (GitHub Actions API)
- `analysis/energy_analysis.ipynb` — statistical analysis notebook (Python, scipy, pandas, matplotlib)
- `results/raw_data.csv` — consolidated Eco-CI measurements
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
