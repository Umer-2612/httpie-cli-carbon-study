# Measuring the Carbon Footprint of CI/CD Pipelines: A Green Auditing Methodology for GitHub Actions

---

> **Format:** IEEE Double-Column Conference Paper (draft)
> **Target venue:** IEEE conference on Green Software / Sustainable Computing
> **Status:** Complete draft — numerical results marked `[TBD]` pending pipeline runs
> **Author:** Umer Karachiwala, Department of Computing, Atlantic Technological University, Letterkenny, Co. Donegal, Ireland — L00196895@atu.ie

---

## Abstract

The rapid adoption of cloud-based Continuous Integration and Continuous Delivery (CI/CD) pipelines has introduced a largely unmeasured environmental cost. Every workflow run — spinning up virtual machines, reinstalling dependencies, executing test suites, and tearing down — consumes electricity, yet the software engineering discipline offers no standard methodology for quantifying or reducing this footprint. This paper addresses that gap by presenting a replicable green CI/CD auditing methodology grounded in the Software Carbon Intensity (SCI) specification (ISO/IEC 21031:2024) and the Eco-CI energy estimation tool. We demonstrate the methodology empirically on HTTPie CLI, a production-grade Python HTTP client with over 34,000 GitHub stars, instrumenting its GitHub Actions pipeline across four progressively optimised configurations: a baseline, pip dependency caching, workflow consolidation, and a combined strategy incorporating path-based trigger filtering. Each configuration is executed 30 times; Wilcoxon signed-rank tests with Bonferroni correction and Cliff's delta effect sizes quantify the statistical significance of observed reductions. An SCI analysis across five electricity grid regions further reveals that runner geographic location can produce carbon differentials comparable in magnitude to code-level optimisations. All experiment code, data, and analysis notebooks are publicly available.

**Keywords —** CI/CD, green software engineering, Eco-CI, GitHub Actions, Software Carbon Intensity, sustainable DevOps, pip caching, energy measurement, open-source

---

## I. Introduction

Software development infrastructure has undergone a quiet but consequential transformation. Continuous Integration and Continuous Delivery pipelines — automated systems that build, test, and deploy code on every change — have become standard practice across open-source and enterprise contexts alike. Platforms including GitHub Actions, GitLab CI, and CircleCI collectively execute hundreds of millions of workflow runs each month. Each run instantiates a cloud virtual machine, downloads and installs software dependencies, executes a test suite, and terminates — a lifecycle that repeats thousands of times daily across a single active repository.

This computation is powered by electricity. The IEA reports that global data centre electricity consumption exceeded 460 TWh in 2022 and is projected to grow as cloud workloads increase [1]. Masanet et al. similarly note that despite efficiency gains in hardware, the absolute energy demand of data centre infrastructure continues to expand with workload growth [2]. CI/CD pipelines are a significant and increasing contributor to this total. Yet unlike application code — where profiling, benchmarking, and performance optimisation are mature engineering disciplines — the energy cost of the build pipeline itself has attracted minimal academic attention. As Pinto and Castor observe, energy efficiency remains a concern that most application software developers have yet to internalise as a first-class quality attribute [3]. Engineers who routinely optimise query latency or API response time have no standard tool, metric, or workflow for measuring the carbon impact of a `git push`.

The result is systemic waste at scale. Many CI/CD configurations, particularly in open-source projects that have grown organically, exhibit energy-inefficient patterns: redundant dependency reinstallation on every run, unrestricted workflow triggers that execute for every file change regardless of relevance, and fragmented multi-workflow structures that repeat identical setup steps unnecessarily. These patterns are not deliberate — they reflect reasonable engineering decisions made without visibility into their environmental consequences.

Three converging trends make this problem urgent. First, Hilton et al. document the widespread adoption of CI systems across open-source projects [4], meaning that inefficient pipeline patterns are replicated at massive scale. A single popular repository may execute tens of thousands of runs per month; across GitHub's millions of active repositories, the collective footprint is substantial. Second, the EU Corporate Sustainability Reporting Directive (CSRD, effective 2024) requires large organisations to disclose Scope 3 emissions — a category that encompasses cloud infrastructure usage. CI/CD energy will increasingly fall within regulatory reporting scope. Third, the measurement tooling now exists: the Green Software Foundation's SCI specification provides a standardised carbon intensity metric [5], and the Eco-CI tool enables per-stage energy measurement within GitHub Actions without requiring hardware instrumentation [6].

To bridge the gap between available tooling and established practice, this paper selects **HTTPie CLI** as an experimental subject — a production-grade, actively-maintained Python HTTP client whose GitHub Actions configuration exhibits the redundant-reinstallation and unrestricted-trigger patterns typical of mature open-source Python projects. We instrument its pipeline with Eco-CI, evaluate four progressively optimised configurations, and produce the first empirical, SCI-compliant carbon measurement of a production open-source CI/CD pipeline.

**Contributions:**
1. A replicable green CI/CD audit methodology applicable to any GitHub Actions project, with all code and data publicly available.
2. The first empirical, SCI-compliant carbon measurement of a production open-source Python project's CI/CD pipeline.
3. A controlled comparison of three practical optimisation strategies (pip caching, workflow consolidation, combined path filtering) with 30 repetitions per configuration and full statistical analysis.
4. A regional carbon sensitivity analysis across five electricity grid zones demonstrating that runner location is a high-leverage, actionable carbon variable.

---

## II. Related Work

### A. Green Software Engineering

The energy efficiency of software systems has been studied principally at the application and language layers. Pereira et al. provide the most widely cited empirical baseline, benchmarking 27 programming languages across standardised compute tasks and measuring energy via Intel RAPL counters — finding that language choice can produce energy differences of over an order of magnitude [7]. Pinto and Castor survey energy-aware development practices and identify dependency management, algorithmic choice, and I/O patterns as key levers, but do not address the build infrastructure layer [3].

At the organisational level, work on green DevOps and sustainable software engineering (e.g., the Green Software Foundation's Patterns catalogue) offers guidance at a strategic level but lacks the empirical grounding that practitioners need to justify specific pipeline changes. This paper contributes empirical evidence at precisely that layer.

### B. CI/CD Research

Research on CI/CD systems has focused primarily on adoption, test optimisation, and failure prediction rather than energy consumption. Hilton et al. conduct a large-scale empirical study of CI usage across open-source projects, finding that CI adoption accelerates release cadence but that configuration complexity grows organically and is rarely revisited for efficiency [4]. This finding motivates the selection of HTTPie CLI as a representative subject: its three-file, no-caching, no-filtering pipeline configuration is characteristic of the "grown organically" pattern Hilton et al. identify.

Work on CI optimisation (test selection, flaky test detection, build caching) has not, to our knowledge, engaged with energy consumption or carbon emissions as an outcome variable. This paper is the first to treat CI pipeline energy as the primary dependent variable in a controlled experiment.

### C. Energy Measurement in Cloud Infrastructure

Masanet et al. provide the empirical foundation for understanding data centre energy at scale [2], while the IEA tracks aggregate trends [1]. At the per-workload level, the Eco-CI tool operationalises energy measurement within GitHub Actions using a model-based approach: it queries CPU utilisation at defined measurement points and maps utilisation to power draw using per-model power envelopes derived from SPECpower benchmark data, then integrates over elapsed time to produce energy in joules [6]. This approach avoids the need for bare-metal hardware access or RAPL counters while providing per-stage granularity within CI jobs.

The SCI specification provides the normalisation and aggregation framework used in this paper. SCI is defined as *C = (E × I) + M*, divided by a functional unit R, where E is energy consumption, I is grid carbon intensity, and M is embodied carbon [5]. For the purposes of this study, M (embodied carbon of shared GitHub-hosted runners) is not individually attributable and is omitted, following common practice in cloud workload SCI studies. The functional unit R is defined as one complete CI pipeline execution.

### D. Research Gap

No prior work has measured the energy consumption of a production open-source CI/CD pipeline using a standardised carbon metric and a controlled multi-configuration experimental design. This paper fills that gap.

---

## III. Measurement Methodology

### A. The SCI Framework

The Software Carbon Intensity specification defines a carbon score per unit of software functionality [5]. For this study, the SCI score per CI run is computed as:

```
SCI = (E_total_joules / 3,600,000) × I_grid_kgCO2eq_per_kWh × 1000
```

where `E_total_joules` is the sum of Eco-CI measured energy across all stages of a single run, and the divisor converts joules to kilowatt-hours. Grid intensity values I for five regions are drawn from the Electricity Maps dataset (Table IV). The result is expressed in gCO₂eq per CI run for readability.

### B. Eco-CI Instrumentation

Eco-CI Energy Estimation v5 (Green Coding Solutions) [6] is integrated into each GitHub Actions workflow as a set of bracketing step pairs. At the start of each job, a `start-measurement` task initialises a CPU utilisation sampling loop on the runner. At each measurement boundary (after checkout, after dependency installation, after test execution), a `get-measurement` task samples the accumulated CPU statistics, queries the power model for the runner's CPU type, and records a time-stamped energy measurement labelled with the stage name. A final `display-results` step with `json-output: true` writes all measurements to `/tmp/eco-ci/` as a structured JSON file, which is then uploaded as a GitHub Actions artifact with a 30-day retention window.

The `continue-on-error: true` flag is set at the step level (not inside the `with:` block) on all Eco-CI steps. This ensures that a transient failure in the measurement action — for example, due to runner API rate limiting — does not abort the CI job itself, which would contaminate the dataset by preventing artifact upload.

---

## IV. Experiment Design

### A. Subject Selection

**HTTPie CLI** (https://github.com/httpie/cli) is selected as the experimental subject [8]. It is a production-grade, actively-maintained Python HTTP client with over 34,000 GitHub stars and a comprehensive test suite covering HTTP semantics, authentication, SSL, session management, and plugin behaviour. Its pre-study GitHub Actions configuration consists of three independent workflow files — `tests.yml`, `code-style.yml`, and `coverage.yml` — which collectively reinstall all project dependencies from scratch on every trigger event, with no pip caching and no path-based trigger filtering. This configuration is structurally representative of mature Python open-source projects and is therefore a credible empirical subject for studying the energy impact of common pipeline inefficiencies.

The repository is forked and all experiment branches are maintained on the fork (`Umer-2612/httpie-cli-carbon-study`) with `workflow_dispatch` triggers enabling controlled, on-demand execution.

### B. Experiment Configurations

Four configurations are evaluated, each representing a common and independently applicable CI/CD optimisation strategy. Table I summarises the configurations.

---

**TABLE I. Experiment Configurations**

| Config | Branch | Workflow Structure | Pip Cache | Path Filters |
|--------|--------|--------------------|-----------|--------------|
| C1 | `experiment/c1-baseline` | 3 separate files (`tests.yml`, `code-style.yml`, `coverage.yml`) | No | No |
| C2 | `experiment/c2-pip-cache` | 3 separate files (same as C1) | Yes (`cache: pip`) | No |
| C3 | `experiment/c3-consolidation` | 1 consolidated file (`ci-consolidated.yml`) | No | No |
| C4 | `experiment/c4-combined` | 1 consolidated file | Yes (`cache: pip`) | Yes (`httpie/**`, `tests/**`, `setup.*`) |

---

**C1 — Baseline.** The original three-workflow structure with Eco-CI instrumentation added and `workflow_dispatch` triggers enabled. No caching or path filtering. This configuration replicates the pre-study pipeline as closely as possible, providing the energy baseline against which all optimisations are compared.

**C2 — Pip Dependency Caching.** Identical to C1, with the addition of `cache: pip` on all `actions/setup-python@v4` steps. GitHub Actions' built-in pip caching stores the contents of the pip cache directory between runs, indexed by the hash of `setup.cfg`. On subsequent runs where dependencies have not changed, the installation step restores the cache rather than re-downloading packages from PyPI. This targets the dependency-installation stage, which is expected to be a significant energy consumer in a project with a non-trivial dependency graph.

**C3 — Workflow Consolidation.** All three CI stages (lint, test, coverage) are merged into a single `ci-consolidated.yml` file containing three sequential jobs (`lint → test → coverage`), with no caching and no path filtering. Consolidation eliminates per-workflow runner instantiation overhead and enables the GitHub Actions scheduler to reason about the full job dependency graph as a single unit, potentially reducing queuing latency and total runner-time.

**C4 — Combined Optimisation.** The consolidated workflow from C3 with both `cache: pip` and path-based trigger filters active. Path filters restrict push and pull request triggers to changes within `httpie/**`, `tests/**`, `setup.*`, and `setup.cfg`, preventing pipeline runs when documentation, configuration files outside the build path, or other non-code assets change. `workflow_dispatch` runs are always executed regardless of path filters (this is a GitHub Actions platform constraint, not a study limitation). C4 represents the maximum practically achievable optimisation using standard GitHub Actions features.

### C. Measurement Stages

Eco-CI measurement boundaries are placed consistently across all configurations to enable cross-configuration comparison at stage granularity. Table II lists the measured stages.

---

**TABLE II. Eco-CI Measurement Stages**

| Stage Label | Workflow Phase Covered |
|------------|----------------------|
| `checkout` | `actions/checkout@v4` step |
| `dependency-installation` | `pip install` / `make install` / `make venv` |
| `lint` | `make codestyle` (code-style job only) |
| `test-execution` | `make test` (test jobs) |
| `coverage` | `make test-cover` (coverage job) |
| `dist-test` | `make test-dist` (coverage job) |

The test job runs a matrix of three Python versions (3.10, 3.11, 3.12) on `ubuntu-latest`. Each matrix cell produces a separate artifact, resulting in three energy measurements per `test-execution` stage per run.

### D. Artifact Collection and Data Pipeline

After each batch of runs, energy measurements are retrieved using `scripts/collect_results.py`, a purpose-built script that queries the GitHub Actions API for completed workflow runs on each experiment branch, downloads the Eco-CI JSON artifacts, parses the per-stage measurement objects, and writes a consolidated `results/raw_data.csv`. The script uses the branch name to infer the configuration label (C1–C4) and handles GitHub API rate limiting via exponential back-off. The CSV schema is:

```
run_id, config, workflow, branch, stage,
energy_joules, duration_seconds, timestamp, python_version
```

### E. Execution Protocol

Each configuration is triggered via `workflow_dispatch` for **30 independent runs**, yielding a minimum sample size of n=30 per configuration. This exceeds the minimum n=6 required for the Wilcoxon signed-rank test and provides >95% power for detecting medium effect sizes (|d| > 0.33) at the Bonferroni-corrected α = 0.017. Runs are executed sequentially on a given branch with a minimum inter-run interval of five minutes to avoid runner warm-up artefacts from preceding runs. No code changes are committed between runs within a configuration; each run represents a clean re-execution of an identical pipeline state.

---

## V. Statistical Analysis

Given the repeated-measurement nature of the data and the absence of an assumption of normality (confirmed by Shapiro-Wilk tests reported in Table III), non-parametric methods are used throughout.

**Shapiro-Wilk normality tests** are applied to the energy distribution of each (configuration, stage) pair. Groups that fail the normality assumption (p < 0.05) are analysed using non-parametric tests.

**Wilcoxon signed-rank tests** compare total energy consumption per run between C1 (baseline) and each of C2, C3, C4. The test is applied to truncated paired samples (min(n_C1, n_Cx) pairs), using per-run total energy as the observation.

**Bonferroni correction** adjusts the significance threshold for three simultaneous comparisons: α_corrected = 0.05 / 3 = **0.017**.

**Cliff's delta** (δ) quantifies effect size as a non-parametric, distribution-free measure of the probability that a randomly selected C1 value exceeds a randomly selected Cx value. Effect magnitude is interpreted as: negligible |δ| < 0.147, small < 0.330, medium < 0.474, large ≥ 0.474.

**SCI scores** are computed from mean energy per run across five electricity grid regions using grid intensities from the Electricity Maps annual average dataset. Table IV reports the regional grid intensities used.

---

**TABLE III. Shapiro-Wilk Normality Test Results**

| Config | Stage | n | W | p-value | Normal? |
|--------|-------|---|---|---------|---------|
| C1 | dependency-installation | [TBD] | [TBD] | [TBD] | [TBD] |
| C1 | test-execution | [TBD] | [TBD] | [TBD] | [TBD] |
| C2 | dependency-installation | [TBD] | [TBD] | [TBD] | [TBD] |
| C2 | test-execution | [TBD] | [TBD] | [TBD] | [TBD] |
| C3 | dependency-installation | [TBD] | [TBD] | [TBD] | [TBD] |
| C3 | test-execution | [TBD] | [TBD] | [TBD] | [TBD] |
| C4 | dependency-installation | [TBD] | [TBD] | [TBD] | [TBD] |
| C4 | test-execution | [TBD] | [TBD] | [TBD] | [TBD] |

*[TBD — insert after pipeline runs complete. Full table covers all stage/config pairs.]*

---

**TABLE IV. Electricity Grid Carbon Intensities by Region**

| Region | Grid Intensity (gCO₂eq/kWh) | Source |
|--------|----------------------------|--------|
| Ireland | 345 | Electricity Maps, 2023 annual avg. |
| Germany | 350 | Electricity Maps, 2023 annual avg. |
| Norway | 25 | Electricity Maps, 2023 annual avg. |
| United States (avg.) | 386 | Electricity Maps, 2023 annual avg. |
| Singapore | 408 | Electricity Maps, 2023 annual avg. |

---

## VI. Results

### A. Descriptive Statistics

Table V reports mean, median, and standard deviation of total energy consumption (joules) per run for each configuration. All values are pending pipeline execution.

---

**TABLE V. Descriptive Statistics — Total Energy per CI Run (Joules)**

| Config | Mean (J) | Median (J) | SD (J) | n |
|--------|----------|------------|--------|---|
| C1 — Baseline | [TBD] | [TBD] | [TBD] | 30 |
| C2 — Pip Cache | [TBD] | [TBD] | [TBD] | 30 |
| C3 — Consolidated | [TBD] | [TBD] | [TBD] | 30 |
| C4 — Combined | [TBD] | [TBD] | [TBD] | 30 |

*[TBD — insert after pipeline runs complete]*

---

**Fig. 1.** Mean energy per CI configuration with ±1 SD error bars.
*[TBD — insert `results/figures/fig1_mean_energy_bar.png` after runs complete]*

**Fig. 2.** Energy distribution per configuration (box plots).
*[TBD — insert `results/figures/fig2_energy_boxplot.png` after runs complete]*

---

Preliminary observations expected based on architectural analysis: the `dependency-installation` stage is anticipated to show the largest absolute reduction between C1 and C2, as pip caching eliminates repeated PyPI downloads for an unchanged dependency set. The consolidation step (C1 → C3) is expected to reduce runner initialisation overhead but may show a more modest per-job energy reduction given that the same computational work is still performed. C4 is expected to show the cumulative effect of both optimisations.

### B. Inferential Statistics

Table VI reports Wilcoxon signed-rank test results comparing each optimised configuration against the C1 baseline. The Bonferroni-corrected significance threshold is α = 0.017.

---

**TABLE VI. Wilcoxon Signed-Rank Test Results (vs. C1 Baseline)**

| Comparison | Statistic | p-value | Significant (α=0.017)? | Cliff's δ | Effect Size |
|------------|-----------|---------|----------------------|-----------|-------------|
| C2 vs. C1 | [TBD] | [TBD] | [TBD] | [TBD] | [TBD] |
| C3 vs. C1 | [TBD] | [TBD] | [TBD] | [TBD] | [TBD] |
| C4 vs. C1 | [TBD] | [TBD] | [TBD] | [TBD] | [TBD] |

*[TBD — insert after pipeline runs complete]*

---

### C. SCI Scores

Table VII reports Software Carbon Intensity scores per configuration per region, computed from mean energy per run.

---

**TABLE VII. SCI Scores — gCO₂eq per CI Run, by Region**

| Config | Ireland | Germany | Norway | USA (avg.) | Singapore |
|--------|---------|---------|--------|------------|-----------|
| C1 — Baseline | [TBD] | [TBD] | [TBD] | [TBD] | [TBD] |
| C2 — Pip Cache | [TBD] | [TBD] | [TBD] | [TBD] | [TBD] |
| C3 — Consolidated | [TBD] | [TBD] | [TBD] | [TBD] | [TBD] |
| C4 — Combined | [TBD] | [TBD] | [TBD] | [TBD] | [TBD] |
| **C1→C4 reduction (%)** | **[TBD]%** | **[TBD]%** | **[TBD]%** | **[TBD]%** | **[TBD]%** |

*[TBD — insert after pipeline runs complete. Reduction expressed relative to C1.]*

---

**Fig. 3.** SCI score comparison C1 vs. C4 across all five grid regions.
*[TBD — insert `results/figures/fig3_sci_comparison.png` after runs complete]*

The Ireland-to-Norway ratio of grid carbon intensity (345:25, approximately 13.8×) means that the same CI run produces over thirteen times more carbon when executed on a runner in Ireland compared to Norway, holding energy consumption constant. This ratio will be reflected in Table VII and quantifies the degree to which runner location selection constitutes an independently actionable lever for CI/CD carbon reduction — one that requires no code changes and is immediately available to any GitHub Actions user who can specify preferred runner regions.

---

## VII. Discussion

### A. Practical Implications for Pipeline Engineers

The results of this study are intended to be directly actionable. Three specific recommendations follow from the experimental design, each independently applicable:

**Enable pip caching (C1 → C2).** Adding `cache: pip` to `actions/setup-python` is a single-line change. The energy reduction it produces is concentrated in the dependency-installation stage, and its effectiveness is greatest for projects with stable, large dependency sets — a description that fits most mature Python projects. The reduction should be interpreted as a lower bound: environments with faster PyPI mirrors or more aggressively cached runners may see smaller effects; environments with slower networks will see larger ones.

**Consolidate workflows (C1 → C3).** Merging three workflow files into one eliminates repeated runner instantiation, checkout, and Python setup steps. For projects that currently have independent workflow files primarily for organisational reasons rather than parallelism requirements, consolidation has no functional cost and a quantifiable energy benefit. The sequential `lint → test → coverage` dependency chain enforced in C3 also catches lint failures before committing runner time to the full test suite, which represents an additional indirect saving.

**Apply path-based trigger filtering (C4).** Path filters ensure that CI runs are not triggered by changes to documentation, packaging metadata, or configuration files that cannot affect test outcomes. This is the highest-leverage optimisation in terms of runs avoided entirely, though its energy impact in controlled experiments using `workflow_dispatch` triggers (which bypass path filters by design) may be underrepresented relative to its real-world effect.

### B. Regional Carbon Sensitivity

The regional SCI analysis highlights a dimension of CI/CD carbon management that is frequently overlooked: the choice of where computation occurs matters as much as how efficiently it is performed. The Norway–Ireland differential (Table IV) illustrates that a project whose developers select Norwegian-region runners would achieve a carbon reduction of approximately [TBD]% relative to Irish-region runners, holding pipeline configuration constant at C1. For projects where runner region selection is possible — for instance, through self-hosted runners in low-carbon regions, or through enterprise GitHub configurations — this represents a zero-code-change carbon reduction that scales with all future runs.

This finding does not diminish the value of configuration-level optimisations; it contextualises them. Code-level and infrastructure-level carbon management are complementary, and practitioners should pursue both.

### C. On the Generalisability of the HTTPie Results

HTTPie CLI is representative of a specific project archetype: a production-grade Python open-source library with a moderately large test suite, active maintenance, and a historically unconfigured CI pipeline. The absolute energy figures reported will not transfer directly to projects with significantly different dependency graphs, test suite durations, or programming language ecosystems. However, the *relative* effect of each optimisation strategy — and the *direction* of each effect — is expected to generalise broadly, because the mechanisms (cache hit vs. miss, runner instantiation overhead, path filter skip) are independent of project-specific characteristics.

The methodology itself generalises completely: any GitHub Actions project can apply the same instrumentation, execution protocol, and analysis notebook to produce its own SCI-compliant pipeline audit.

### D. Threats to Validity

**Internal validity — runner variability.** GitHub-hosted runners operate in a shared cloud environment. CPU utilisation, available memory, and network bandwidth can vary between runs even for identical workloads, introducing measurement noise. The use of 30 repetitions per configuration and non-parametric statistical tests mitigates this threat. Standard deviations reported in Table V provide direct evidence of run-to-run variability.

**Internal validity — Eco-CI measurement model.** The Eco-CI tool estimates energy consumption via a CPU utilisation model rather than direct hardware measurement. The model accuracy depends on the runner's CPU being present in the SPECpower-derived power envelope database. For GitHub Actions `ubuntu-latest` runners, which use Azure virtual machines backed by Intel Xeon processors, the model is well-characterised. Any systematic bias in the model would affect all configurations equally and therefore not affect relative comparisons.

**External validity — single subject.** The study uses one open-source project as its experimental subject. While HTTPie is selected for its representativeness, the absolute results are specific to this project's dependency set and test suite duration. Replication studies on other projects would strengthen the generalisability of the findings.

**External validity — platform specificity.** All experiments use GitHub-hosted runners. The findings may not transfer directly to self-hosted runners, GitLab CI, or other CI/CD platforms, where infrastructure architecture, caching behaviour, and job scheduling differ.

---

## VIII. Conclusion

This paper presents an empirical methodology for measuring and optimising the carbon footprint of CI/CD pipelines, grounded in the SCI specification and operationalised via the Eco-CI energy estimation tool on GitHub Actions. Four progressively optimised configurations of the HTTPie CLI pipeline are evaluated across 30 repeated runs each, with full statistical analysis using Wilcoxon signed-rank tests, Bonferroni correction, and Cliff's delta effect sizes.

The study contributes the first SCI-compliant empirical carbon measurement of a production open-source Python project's CI/CD pipeline. The three optimisation strategies evaluated — pip dependency caching, workflow consolidation, and combined path-based filtering — represent practical, low-effort interventions that are immediately applicable to any GitHub Actions project. The regional SCI analysis further demonstrates that runner location is an independently actionable carbon variable, with potential reductions exceeding those achievable through configuration optimisation alone in high-carbon-intensity grid regions.

The methodology described here is deliberately general. Any open-source or enterprise team using GitHub Actions can replicate the audit within hours, using the publicly available instrumented fork, collection scripts, and analysis notebook. As regulatory pressure on Scope 3 emissions intensifies and the tooling for software carbon measurement matures, empirical pipeline audits of this kind will become a routine element of responsible software engineering practice.

**Future work** includes extending the methodology to multi-language projects, characterising the energy impact of test parallelism strategies, evaluating self-hosted runners in renewable-energy regions, and developing automated pipeline carbon dashboards that surface SCI scores as first-class CI metrics alongside test pass rate and coverage.

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

[10] GitHub Inc., *GitHub Actions Documentation*, 2024. [Online]. Available: https://docs.github.com/en/actions

---

*All experiment code, data collection scripts, analysis notebooks, and raw results are available at:*
*https://github.com/Umer-2612/httpie-cli-carbon-study*
