# College Tour TSP 

This repository contains a constrained, real-world **College Tour Traveling Salesperson Problem (TSP)** solver for Istanbul. The problem involves visiting a set of universities **exactly once**, while organizing visits into **multi-day routes** that start and end at a fixed depot (e.g., **Taksim Meydan**). The solution approach is based on **Adaptive Large Neighborhood Search (ALNS)** enhanced with **Simulated Annealing (SA)** acceptance, and strengthened with **2-opt local search** applied per day.

In addition to the solver, the repository includes an experimental framework and an analysis notebook to evaluate:
- convergence behavior,
- sensitivity to SA hyperparameters,
- robustness across random seeds,
- and overall performance compared to a random baseline.

---

## Problem Definition

### Goal
Minimize the **total travel cost** while generating a feasible and efficient multi-day tour plan.

### Nodes
- **Universities** are nodes with unique IDs (1..58).
- The **depot** node is **Taksim Meydan (node_id = 59)**.
- Each day route starts/ends at the depot.

### Key Constraints
This instance includes constraints commonly seen in realistic routing problems:

**Hard Constraints (must always hold)**
- Each university must be visited **exactly once** (except depot).
- No repeated nodes within the same day route.
- Routes must begin and end at the depot.

**Soft Constraints (allowed with penalties)**
- Daily working time limit (e.g., 8 hours)
- Time windows (arrival/visit constraints)
- Priority universities should be visited earlier (schedule preference)

Soft constraints are modeled through **penalty costs** added to the objective function. This gives flexibility and realism: the algorithm is allowed to violate constraints only if doing so is cheaper than forcing strict compliance.

---

## Objective Function

The algorithm minimizes a composite objective:

**Total Objective = Total Travel Distance/Time + Total Penalty Cost**

Penalty costs may include:
- Daily duration overrun penalties,
- Time-window delay penalties,
- Priority violation penalties.

This design ensures:
- hard feasibility is enforced,
- practical constraints can be traded off realistically.

---

## Algorithm Summary

### 1) Initialization (Cold Start)
A **Cold-Start** strategy is used for unbiased evaluation:
- Non-depot nodes are randomly shuffled,
- distributed across daily routes,
- producing a feasible but high-cost initial solution.

### 2) ALNS Main Loop
Each iteration applies:
- **Destroy operator**: removes a subset of nodes (random removal in the baseline version)
- **Repair operator**: reinserts removed nodes via greedy insertion minimizing incremental cost + penalties
- **Local search**: day-wise **2-opt** optimization

### 3) Acceptance Criterion (Simulated Annealing)
Even worse solutions may be accepted with a probability controlled by:
- Initial Temperature (**Tinitial**)
- Cooling Rate (**α**)
- Temperature update: **T = Tinitial · α^k**

This prevents premature convergence and supports exploration early in the search.

### 4) Logging
Per-iteration logs are stored (e.g., best cost, current cost, feasibility), enabling deep analysis and visualization.

---

## Data and Mapping

### Distance Matrix
The solver expects a square N×N distance matrix (asymmetric allowed), for example:

`/kaggle/input/taksim-distance-matrix/universite_distance_matrix_osrm_km_taksim.csv`

### Node Mapping (Example)
The dataset includes 58 universities + 1 depot (Taksim Meydan = 59).  
A sample mapping is shown below (excerpt):

- 1: Acıbadem Mehmet Ali Aydınlar Üniversitesi  
- 38: İstanbul Teknik Üniversitesi  
- 53: Sabancı Üniversitesi  
- 58: Yıldız Teknik Üniversitesi  
- 59: Taksim Meydan (Depot)

---

## Experimental Results and Evaluation

## CHAPTER 4: EXPERIMENTAL RESULTS AND EVALUATION
This chapter presents experimental results of the ALNS algorithm developed for the constrained College Tour TSP. It details parameter tuning, final performance evaluation against a random baseline, and analysis of the best-found solution.

---

### 4.1 Parameter Tuning Study

A comprehensive parameter tuning study was conducted to improve ALNS performance, focusing specifically on the **Simulated Annealing acceptance criterion** hyperparameters:
- **Initial Temperature (Tinitial)**
- **Cooling Rate (α)**

Although the ALNS framework contains other parameters (destroy–repair operator weights, removal size, local search settings), preliminary experiments showed these had **secondary influence** compared to SA settings. Therefore, they were fixed to reduce tuning dimensionality and to focus on the strongest drivers of convergence and solution quality.

#### 4.1.1 Tuning Parameters and Criteria

**Initial Temperature**
- Tested range: **[300, 3000]**  
- Motivation: higher temperatures increase early exploration and reduce trapping in local minima.

**Cooling Rate**
- Tested range: **[0.95, 0.99]**  
- Motivation: α closer to 1 yields slower cooling and sustained acceptance of worse solutions, which helps avoid premature convergence in constrained instances.

Each parameter configuration was tested with **multiple independent runs** to mitigate stochasticity. Primary metric was **Mean Final Best Cost**. Additional metrics:
- minimum cost,
- standard deviation,
- feasibility rate,
- and mean runtime (cost-efficiency trade-off).

#### 4.1.1.1 Tuning Configurations

**Table 1. Algorithm Tuning Configurations**

| Parameter | Test Values |
|---|---|
| Start temperature (Tstart) | {300, 1000, 3000} |
| Temperature decrease (α) | {0.95, 0.98, 0.99} |

Final temperature is not tuned directly; it is implied by:
**Tfinish = Tstart · α^N**

Final temperatures are reported for transparency to show effective cooling behavior across settings.

#### 4.1.2 Tuning Results and Selection

Tuning outcomes were summarized via a heatmap showing **average final best cost** per (Tinitial, α) combination.

**Table 2. Algorithm Tuning Metrics**

| INITIAL_T | ALPHA_T | Mean Final Cost | Min Cost | Std Dev | Feasible Rate | Mean Runtime (s) |
|---:|---:|---:|---:|---:|---:|---:|
| 300 | 0.99 | 1,271,052 | 1,252,955 | 11,890 | 1.00 | 236 |
| 1000 | 0.99 | 1,272,532 | 1,256,399 | 11,697 | 1.00 | 254 |
| 3000 | 0.99 | 1,273,294 | 1,243,297 | 14,980 | 1.00 | 346 |
| 3000 | 0.98 | 1,282,335 | 1,246,212 | 17,374 | 1.00 | 138 |
| 300 | 0.98 | 1,282,796 | 1,263,257 | 13,974 | 1.00 | 120 |
| 1000 | 0.98 | 1,283,660 | 1,260,190 | 14,954 | 1.00 | 127 |
| 1000 | 0.95 | 1,303,580 | 1,287,370 | 11,635 | 1.00 | 50 |
| 300 | 0.95 | 1,303,766 | 1,290,379 | 10,757 | 1.00 | 68 |
| 3000 | 0.95 | 1,312,372 | 1,284,187 | 19,906 | 1.00 | 54 |

**Selected Configuration**
Based on lowest average final cost without unnecessary computational overhead:
- **Tinitial = 300**
- **α = 0.99**

---

### 4.2 Final Algorithm Performance Evaluation

After selecting the best SA configuration, the algorithm was evaluated under **Cold-Start initialization**. The chosen configuration was executed over **10 independent runs**, each using a different random seed.

#### 4.2.1 Aggregate Run Statistics

**Table 3. Cold Start Summary Statistics**

| Metric | Value |
|---|---:|
| Number of runs | 10 |
| Mean Final Cost | 1,277,486.62 |
| Standard Deviation | 12,190.02 |
| Best Final Cost | 1,255,127.24 |
| Worst Final Cost | 1,292,663.90 |
| Average Runtime (sec) | 294.37 |

These results indicate consistent performance across runs. The relatively low standard deviation demonstrates robustness under stochastic cold-start conditions.

#### 4.2.2 Improvement vs Random Baseline

To quantify effectiveness, results were compared against the average **Initial Cost** obtained from random route assignment (pre-ALNS).

**Table 4. Optimization Baseline and Improvement**

| Metric | Value |
|---|---:|
| Mean Initial Cost | 2,308,901.91 |
| Minimum Initial Cost | 2,223,235.46 |
| Maximum Initial Cost | 2,421,774.30 |
| Initial Cost Std. Dev. | 66,280.64 |
| ALNS Best Final Cost | 1,255,127.24 |
| ALNS Improvement Percentage | 45.64% |

Random initialization yields high cost and high instability, while ALNS achieves ~**45.64% improvement** compared to this baseline.

#### 4.2.3 Consistency Analysis (Gap)

Since no known optimum exists for this custom real-world instance, the **best-found solution** across runs is used as reference.

Gap is computed as:

Gap = 100 × (Mean Final Cost − Best Final Cost) / Best Final Cost  
≈ 100 × (1,277,486.62 − 1,255,127.24) / 1,255,127.24  
≈ **1.78%**

This gap reflects the average deviation from the best-known solution and serves as a meaningful robustness indicator.

#### 4.2.4 Local Optimality Validation (Warm Start)

To test if the best-found solution can be improved further, a Warm Start experiment was performed:
- The best cold-start solution was used as the initial solution for new ALNS runs.

Warm-start results produced **no significant improvement**, suggesting the solution is located in a deep high-quality local basin, and that either:
- stronger perturbation operators, or
- alternative cooling schedules / neighborhood moves  
would be needed to escape toward a better global region.

---

### 4.3 Detailed Analysis of the Best Solution

The best solution provides a complete 20-day schedule visiting all 58 universities.

**Table 5. Best Solution Route Schedule (Summary)**

| node_id | University |
|---:|---|
| 1 | Acıbadem Mehmet Ali Aydınlar Üniversitesi |
| 2 | Altınbaş Üniversitesi |
| 3 | ATAŞEHİR ADIGÜZEL MESLEK YÜKSEKOKULU |
| 4 | BAHÇEŞEHİR ÜNİVERSİTESİ |
| 5 | BEYKOZ ÜNİVERSİTESİ |
| 6 | BEZM-İ ÂLEM VAKIF ÜNİVERSİTESİ |
| 7 | BİRUNİ ÜNİVERSİTESİ |
| 8 | BOĞAZİÇİ ÜNİVERSİTESİ |
| 9 | DEMİROĞLU BİLİM ÜNİVERSİTESİ |
| 10 | DOĞUŞ ÜNİVERSİTESİ |
| 11 | FATİH SULTAN MEHMET VAKIF ÜNİVERSİTESİ |
| 12 | FENERBAHÇE ÜNİVERSİTESİ |
| 13 | GALATASARAY ÜNİVERSİTESİ |
| 14 | HALİÇ ÜNİVERSİTESİ |
| 15 | IŞIK ÜNİVERSİTESİ |
| 16 | İBN HALDUN ÜNİVERSİTESİ |
| 17 | İSTANBUL 29 MAYIS ÜNİVERSİTESİ |
| 18 | İSTANBUL AREL ÜNİVERSİTESİ |
| 19 | İSTANBUL ATLAS ÜNİVERSİTESİ |
| 20 | İSTANBUL AYDIN ÜNİVERSİTESİ |
| 21 | İSTANBUL BEYKENT ÜNİVERSİTESİ |
| 22 | İSTANBUL BİLGİ ÜNİVERSİTESİ |
| 23 | İSTANBUL ESENYURT ÜNİVERSİTESİ |
| 24 | İSTANBUL GALATA ÜNİVERSİTESİ |
| 25 | İSTANBUL GEDİK ÜNİVERSİTESİ |
| 26 | İSTANBUL GELİŞİM ÜNİVERSİTESİ |
| 27 | İSTANBUL KENT ÜNİVERSİTESİ |
| 28 | İSTANBUL KÜLTÜR ÜNİVERSİTESİ |
| 29 | İSTANBUL MEDENİYET ÜNİVERSİTESİ |
| 30 | İSTANBUL MEDİPOL ÜNİVERSİTESİ |
| 31 | İSTANBUL NİŞANTAŞI ÜNİVERSİTESİ |
| 32 | İSTANBUL OKAN ÜNİVERSİTESİ |
| 33 | İSTANBUL RUMELİ ÜNİVERSİTESİ |
| 34 | İSTANBUL SABAHATTİN ZAİM ÜNİVERSİTESİ |
| 35 | İSTANBUL SAĞLIK VE SOSYAL BİLİMLER MESLEK YÜKSEKOKULU |
| 36 | İSTANBUL SAĞLIK VE TEKNOLOJİ ÜNİVERSİTESİ |
| 37 | İSTANBUL ŞİŞLİ MESLEK YÜKSEKOKULU |
| 38 | İSTANBUL TEKNİK ÜNİVERSİTESİ |
| 39 | İSTANBUL TİCARET ÜNİVERSİTESİ |
| 40 | İSTANBUL TOPKAPI ÜNİVERSİTESİ |
| 41 | İSTANBUL ÜNİVERSİTESİ |
| 42 | İSTANBUL ÜNİVERSİTESİ-CERRAHPAŞA |
| 43 | İSTANBUL YENİ YÜZYIL ÜNİVERSİTESİ |
| 44 | İSTİNYE ÜNİVERSİTESİ |
| 45 | KADİR HAS ÜNİVERSİTESİ |
| 46 | KOÇ ÜNİVERSİTESİ |
| 47 | MALTEPE ÜNİVERSİTESİ |
| 48 | MARMARA ÜNİVERSİTESİ |
| 49 | MEF ÜNİVERSİTESİ |
| 50 | MİMAR SİNAN GÜZEL SANATLAR ÜNİVERSİTESİ |
| 51 | ÖZYEĞİN ÜNİVERSİTESİ |
| 52 | PİRİ REİS ÜNİVERSİTESİ |
| 53 | SABANCI ÜNİVERSİTESİ |
| 54 | SAĞLIK BİLİMLERİ ÜNİVERSİTESİ |
| 55 | TÜRK-ALMAN ÜNİVERSİTESİ |
| 56 | ÜSKÜDAR ÜNİVERSİTESİ |
| 57 | YEDİTEPE ÜNİVERSİTESİ |
| 58 | YILDIZ TEKNİK ÜNİVERSİTESİ |
| 59 | Taksim Meydan (Depot) |


| Day | Start/End (Node Route) | #Visited Universities | Total Route Distance (km) | Total Travel Time (hours) | #Priority Universities |
|---:|---|---:|---:|---:|---:|
| 1  | 59-41-58-13-59 | 3 | 18.97  | 6.63 | 3 |
| 2  | 59-26-23-42-59 | 3 | 73.47  | 8.45 | 1 |
| 3  | 59-55-30-59    | 3 | 54.98  | 7.83 | 3 |
| 4  | 59-56-54-48-59 | 3 | 35.49  | 7.18 | 1 |
| 5  | 59-2-16-59     | 2 | 59.00  | 5.97 | 0 |
| 6  | 59-50-38-46-59 | 3 | 54.99  | 7.83 | 3 |
| 7  | 59-51-10-3-59  | 3 | 73.66  | 8.46 | 1 |
| 8  | 59-6-43-44-59  | 3 | 21.88  | 6.73 | 0 |
| 9  | 59-32-53-22-59 | 3 | 118.16 | 9.94 | 1 |
| 10 | 59-35-45-24-59 | 3 | 16.53  | 6.55 | 0 |
| 11 | 59-18-33-59    | 2 | 190.58 | 10.35 | 0 |
| 12 | 59-57-47-25-59 | 3 | 78.17  | 8.61 | 0 |
| 13 | 59-9-49-37-59  | 3 | 26.93  | 6.90 | 0 |
| 14 | 59-11-7-40-59  | 3 | 21.95  | 6.73 | 0 |
| 15 | 59-27-19-39-59 | 3 | 21.40  | 6.71 | 0 |
| 16 | 59-34-20-28-59 | 3 | 53.93  | 7.80 | 0 |
| 17 | 59-21-31-15-59 | 3 | 37.94  | 7.26 | 0 |
| 18 | 59-14-22-36-59 | 3 | 23.71  | 6.79 | 0 |
| 19 | 59-4-17-5-59   | 3 | 44.09  | 7.47 | 0 |
| 20 | 59-29-1-12-59  | 3 | 43.79  | 7.47 | 0 |
| **Total** | - | **58** | **1069.61** | **151.65** | **13** |




#### 4.3.1 Constraint Management via Soft Penalties

Daily working time (8h) and time-window constraints are treated as **soft**. Therefore, limited violations may occur if the penalty cost is smaller than the additional travel cost required for strict compliance.

Observed daily time overruns on Days 9, 11, and 12 are not errors; they are deliberate outcomes of penalty-based optimization.

#### 4.3.2 Penalty Interpretation

Difference between reported final objective and total route distance corresponds to total penalty:

Total Penalty Cost  
= Reported Cost − Calculated Route Distance  
≈ 1,255,127.24 m − 1,069,610.00 m  
≈ **185,517.24 m**

This penalty mainly aggregates:
- Daily duration overrun penalties
- Priority violation penalties
- Time window delay penalties

---

# ALNS Algorithm Performance Analysis & Parameter Tuning Visualization

## Project Overview
This notebook is designed to analyze and visualize the performance metrics of the **Adaptive Large Neighborhood Search (ALNS)** algorithm applied to the Traveling Salesperson Problem (TSP). The study specifically focuses on the impact of the **Simulated Annealing (SA)** mechanism and its hyperparameters on the optimization process.

Comprehensive Exploratory Data Analysis (EDA) was conducted using **R** and various libraries to understand the algorithm's behavior under various configurations.

## Analysis Objectives
This study aims to answer the following key questions:
- How fast does the algorithm converge to a global (or near-global) optimum?
- What is the impact of the **Cooling Rate (α)** and **Initial Temperature (T)** on the final solution quality?
- How stable is the algorithm across different random seeds?

## Visualizations & Content
This notebook includes the following visual analyses:

### 1. Convergence Behavior
- **Optimization Progress:** Tracking the descent of the objective function (Total Cost) over iterations.
- **Cooling Schedule:** A dual-axis visualization showing the relationship between Temperature (T) decay and the improvement of the best solution found.

### 2. Parameter Tuning
- **Cost Comparison:** Comparing average objective function values across different combinations of T and α.
- **Interaction Plots:** Analyzing interaction effects between initial temperature and cooling rate.

### 3. Stability Analysis
- **Boxplot Analysis:** Visualizing distribution, median, and variance (IQR) of solutions for each parameter set.
- **Performance Heatmap:** Matrix view of average costs to identify the parameter “sweet spot.”
