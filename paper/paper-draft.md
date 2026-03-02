# Measuring and Optimising Carbon Emissions in CI/CD Pipelines: An Empirical Study Using GitHub Actions

**[IEEE Conference Paper — Draft]**

---

## Abstract

The rapid adoption of cloud-based Continuous Integration and Continuous Delivery (CI/CD) pipelines has introduced a hidden and largely unmeasured environmental cost. Every workflow run consumes energy — spinning up virtual machines, installing dependencies, executing tests, and tearing down — yet software engineering practice offers no standard methodology for quantifying or reducing this footprint. This paper addresses that gap. We propose a replicable green CI/CD auditing methodology grounded in the Software Carbon Intensity (SCI) specification (ISO/IEC 21031:2024) and the Eco-CI energy estimation tool, and we demonstrate it empirically on a real, production-grade open-source project. Using HTTPie CLI — a widely-used Python HTTP client with 34k+ GitHub stars — as a representative subject, we instrument its GitHub Actions pipeline and evaluate four progressively optimised configurations: a baseline, pip dependency caching, workflow consolidation, and a combined strategy incorporating path-based trigger filtering. Each configuration is executed 30 times; Wilcoxon signed-rank tests with Bonferroni correction and Cliff's delta effect sizes quantify the significance of observed reductions. A five-region SCI analysis further shows that the geographic location of CI runners is a high-leverage variable that can produce carbon differentials comparable to code-level optimisations. Our results provide the first empirical, SCI-compliant carbon measurement of production open-source CI/CD infrastructure and establish actionable, evidence-based guidelines for green pipeline engineering.

**Keywords** — CI/CD, green software engineering, Eco-CI, GitHub Actions, Software Carbon Intensity, sustainable DevOps, pip caching, open-source software, carbon emissions

---

## 1. Introduction

### 1.1 The Hidden Carbon Cost of CI/CD

Software development has undergone a quiet infrastructure revolution. Continuous Integration and Continuous Delivery pipelines — automated systems that build, test, and deploy code on every change — have become standard practice across open-source and enterprise organisations alike. Platforms such as GitHub Actions, GitLab CI, and CircleCI collectively execute hundreds of millions of workflow runs every month. Each run instantiates a cloud virtual machine, downloads and installs software dependencies, executes a test suite, and terminates — a lifecycle that repeats thousands of times daily across a single active project.

This computation is powered by electricity. The International Energy Agency reports that global data centre electricity consumption exceeded 460 TWh in 2022 and is projected to grow substantially as cloud workloads increase [1]. CI/CD pipelines are a significant and growing contributor to this total. Yet unlike application code — where profiling, benchmarking, and performance optimisation are mature disciplines — the energy cost of the build pipeline itself has attracted minimal academic attention. Engineers who routinely optimise query latency or API response times have no standard tool, metric, or workflow for measuring the carbon impact of a `git push`.

The result is systemic waste at scale. Many CI/CD configurations, particularly in open-source projects that have grown organically over time, exhibit patterns that are inefficient from an energy perspective: redundant dependency reinstallation on every run, unrestricted workflow triggers that execute for every file change regardless of relevance, and fragmented multi-file workflow structures that repeat setup steps unnecessarily. These patterns are not deliberate — they reflect reasonable engineering decisions made without visibility into their environmental consequences.

### 1.2 Why This Problem Matters Now

The urgency of this problem is driven by three converging trends. First, the sheer scale of CI/CD adoption means that even small per-run inefficiencies compound into substantial aggregate energy consumption. A single popular GitHub repository might execute tens of thousands of workflow runs per month; multiply this across the millions of active repositories on GitHub alone, and the collective footprint is considerable.

Second, regulatory and institutional pressure on software organisations to report and reduce their carbon emissions is increasing. The EU Corporate Sustainability Reporting Directive (CSRD), effective from 2024, requires large organisations to disclose their Scope 3 emissions, which include cloud infrastructure usage. CI/CD energy consumption will increasingly fall within scope.

Third, the tools to address this problem now exist. The Green Software Foundation's SCI specification provides a standardised carbon intensity metric for software systems [2]. The Eco-CI Energy Estimation tool enables direct per-stage energy measurement within GitHub Actions workflows without requiring external hardware instrumentation [3]. The convergence of standard metrics and accessible measurement tooling creates, for the first time, a practical pathway from intention to action for green CI/CD engineering.

### 1.3 The Gap in Existing Research

Existing research on green software engineering has focused predominantly on application-layer optimisation — reducing the energy consumption of running software through algorithmic efficiency, data structure selection, or language choice [4][5]. The CI/CD pipeline layer has received comparatively little attention. Studies that do address build system energy consumption typically rely on synthetic benchmarks or laboratory-controlled environments, limiting their ecological validity and practical applicability.

This paper argues that empirical measurement of real, production CI/CD pipelines is both feasible and essential. Real pipelines exhibit the authentic inefficiency patterns that synthetic benchmarks cannot replicate. Real dependency graphs, real test suites, and real trigger configurations produce measurements that practitioners can directly interpret and act upon.

### 1.4 Our Approach: Empirical Measurement on a Production Project

To bridge this gap, we select **HTTPie CLI** as our experimental subject. HTTPie is a production-grade, actively-maintained open-source Python HTTP client with over 34,000 GitHub stars. Its GitHub Actions configuration — at the time of study — consists of three independent workflow files that collectively reinstall all project dependencies from scratch on every trigger event, with no caching and no path filtering. This configuration is structurally typical of mature Python open-source projects, making it a highly representative and credible empirical subject.

We fork the repository and instrument it with Eco-CI, then systematically introduce and measure three optimisations — pip dependency caching, workflow consolidation, and combined path-based filtering — across 30 repeated runs per configuration. By grounding our measurements in SCI and evaluating them across five electricity grid regions, we produce results that are both statistically rigorous and practically interpretable.

Crucially, the HTTPie experiment is a vehicle, not the destination. The methodology we develop is general: any open-source or enterprise team using GitHub Actions can apply it to audit and improve their own pipeline within hours.

### 1.5 Contributions

The contributions of this paper are:

1. **A replicable green CI/CD audit methodology** based on Eco-CI and the SCI specification, applicable to any project using GitHub Actions, with all experiment code, data, and analysis notebooks publicly available.

2. **The first empirical, SCI-compliant carbon measurement** of a production open-source Python project's CI/CD pipeline, providing a concrete baseline against which optimisations can be assessed.

3. **A controlled experimental comparison** of three practical optimisation strategies — pip caching, job consolidation, and combined path filtering — with 30 repetitions per configuration, Wilcoxon signed-rank testing, Bonferroni correction, and Cliff's delta effect sizes.

4. **A regional carbon sensitivity analysis** across five electricity grid zones (Ireland, Germany, Norway, United States, Singapore), demonstrating that runner geographic location is a high-leverage carbon variable that practitioners can influence through runner selection alone.

5. **Evidence-based guidelines** for green CI/CD engineering, derived from empirical measurement rather than theoretical modelling.

### 1.6 Paper Structure

The remainder of this paper is structured as follows. Section 2 surveys related work in green software engineering, CI/CD optimisation, and energy measurement. Section 3 describes the experimental design, the four pipeline configurations, and the measurement methodology. Section 4 presents quantitative results and statistical analysis. Section 5 discusses the implications for practitioners, open-source maintainers, and platform providers. Section 6 concludes and identifies directions for future work.

---

## References

[1] International Energy Agency, *Data Centres and Data Transmission Networks*, IEA, 2023. [Online]. Available: https://www.iea.org/energy-system/buildings/data-centres-and-data-transmission-networks

[2] Green Software Foundation, *Software Carbon Intensity (SCI) Specification — ISO/IEC 21031:2024*, GSF, 2024. [Online]. Available: https://greensoftware.foundation/projects/software-carbon-intensity

[3] L. Hilpert, M. Fincke, and A. Guldner, *Eco-CI: Measuring the Energy Consumption of CI/CD Pipelines*, Green Coding Solutions, 2023. [Online]. Available: https://github.com/green-coding-solutions/eco-ci-energy-estimation

[4] *(Related work — to be completed)*

[5] *(Related work — to be completed)*
