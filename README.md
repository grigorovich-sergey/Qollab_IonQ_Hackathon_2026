![](https://pub.qollab.xyz/assets/01a12cc6-db78-78b5-a566-8f8ca401049c)

# Road-Maintenance Scheduling with QAOA

*A complete workflow from OpenStreetMap road networks to quantum optimization of road maintenance scheduling*

## Status

This repository is a complimentary/supplementary to the Qollab project [Road-Maintenance Scheduling with QAOA](https://qollab.xyz/u/sergey_gris/road-closure-planning)

The project was submitted to the Qollab x IonQ Quantum hackathon, October 2026 [Hackathon](https://qollab.xyz/programs/hackathon)

## Summary

Road maintenance scheduling is not just about which roads to repair, but which roads can be closed together. This project explores quantum optimization for coordinating simultaneous road closures, using a real road network in Greater Montréal to minimize maintenance time while avoiding excessive detours along an important route.

Despite the tight hackathon timeframe, this prototype demonstrates a **complete end-to-end workflow**: extracting a real road network, identifying conflicting closures, translating them into a QUBO, and evaluating QAOA-generated schedules against an exact classical reference. The deliberately small scheduling instance was tested using **ideal Qiskit Aer** and the **IonQ Forte noisy simulator**.

**Project video presentaion:** [Youtibe](https://www.youtube.com/watch?v=juUiEh5kwfA)


## Running the experiment

**A short execution will not reproduce the reported statistical results.** The full IonQ Forte noisy-simulator experiment requires a substantial sampling budget of 10,000+ raw shots.

For a quick verification, judges can use the lightweight preview mode to check the original input files and SHA-256 checksums, subject to final validation of this mode. The recorded results, figures, and report provide the experimental evidence.

While some files can be found in Qollab project repository, the larger files (exceeding the size limits), as well as precomputing notebooks, can be found on complimentary GitHub repo.

The IonQ experiment runs on the 29-qubit IonQ Forte noise-model simulator, not on quantum hardware.

## 1. Problem Statement

Road maintenance is a familiar challenge in any large city. Roads, bridges, and intersections need regular repairs, but closing them inevitably disrupts traffic. As a Montrealer, I see this problem firsthand: construction projects often cause significant inconvenience, particularly when several important roads are blocked at the same time.

While some closures have only local effects, certain combinations can eliminate convenient alternative routes or produce substantial detours, disrupting important connections across the network. So, the big part of the scheduling challenge is a decision **which roads can be closed together**.

To simplify, it comes down to a **question**: How can a backlog of maintenance jobs be completed as quickly as possible without closing mutually disruptive roads at the same time?

I wanted to explore a quantum optimization approach to this problem of road-maintenance scheduling. Given the limited hackathon timeframe, I narrowed the experiment to one real region: the Châteauguay–Kahnawake area. I have focused on two key landmarks: **a critical destination point - Hôpital Anna-Laberge**, and **a major access point - Honoré-Mercier Bridge**, which connects the are to the Montréal Island.

The objective was to preserve acceptable travel-time connectivity between these points while scheduling a small set of maintenance jobs. In this simplified model, closures were considered unacceptable if they disconnected the selected trip or caused the detour increasing the fastest available travel time by more than **20%**.

The study uses estimated free-flow travel times rather than measured congestion or emergency response times. It is a focused scheduling demonstration, not a complete model of urban transportation or hospital accessibility.

![Figure 1 — Region of interest - the Châteauguay–Kahnawake area](https://pub.qollab.xyz/assets/01a12a75-f3c2-7b96-901f-590948c5e1b3)

## 2. Related Work

Quantum optimization has previously been explored for transportation-network problems, including the network-design study by Dixit and Niu (2023). This project considers a narrower problem: scheduling compatible road closures using a QUBO formulation and gate-based QAOA.

The implementation also draws on the analytical single-layer QAOA results of Ozaeta et al. (2022) for parameter initialization.

## 3. From Road Closures to Quantum Scheduling

The project started with the **area of interest** selection on an interactive Folium map in the Colab notebook. The resulting road network was then downloaded from OpenStreetMap using OSMnx, and converted into a **NetworkX graph**, preserving *travel time, speed limits and directionality*. I first established the fastest route between the selected endpoints, then examined what happened when different road segments were closed.

![Figure 2.A - Road network graph with Origin (Honoré-Mercier Bridge) and Destination (Hôpital Anna-Laberge)](https://pub.qollab.xyz/assets/01a12aad-9cf9-7a45-8f0f-2171dcc02344)
![Figure 2.B - Road network graph with critical singles and pair on detours](https://pub.qollab.xyz/assets/01a12a81-bb5b-72a8-a0cb-5cbb311a9a54)

Some closures could be accommodated by alternative routes, while others caused excessive detours. Road segments that cause substantial disruptions when closed, were marked as sole-critical (red on the graph) and are not analyzed directly, but can not be a part of critical pairs.

More interestingly, certain roads that could be closed individually became problematic when closed together (yellow segments on the graph). A pair of road segments that results in unacceptable detour time, if closed together, is marked as paired-critical. These observations were recorded as scheduling conflicts – they should not be included in a maintenance plan in the same period.

The analysis identified 11 critical single closures and 102 critical pairs across the study network. Each critical pair is two individually acceptable closures whose simultaneous occurrence should be avoided. A road segment can participate in multiple such pairs.

With these conflicts identified, the next question was how to organize maintenance work. For a small demonstration, I selected 10 road segments that have multiple classical non-trivial solutions for 5 periods, assuming each can be completed in one scheduling period.

Instead of assigning every job to a particular time slot, the scheduling problem was represented through compatible pairs of jobs.

This produced a compact optimization problem suitable for QAOA.

![Figure 2 — Workflow: OpenStreetMap → closure analysis → scheduling constraints → QUBO → QAOA sampling → evaluation.](https://pub.qollab.xyz/assets/01a12cc5-069c-7058-9aa1-1849cfa2fe10)

## 4. Implementation: From Scheduling to QAOA

The demonstration uses 10 maintenance jobs (road segments) and equal-duration work periods. All jobs must be completed exactly once, and the objective is to minimize the total number of periods. I assume that two crews are available for every period.

A straightforward formulation would assign every job to a crew and a time slot, requiring many binary variables. Instead, each variable in my formulation represents a compatible pair of jobs that can be completed together.

Of the 45 possible pairs of ten jobs, 20 were forbidden by the closure constraints and 25 were compatible, giving a 25-variable QUBO:

$$
E(x) = -\sum_a x_a + 2\sum_{(a,b)\in O}x_a x_b
$$

Here, each binary variable indicates whether a compatible pair is selected, while O contains pairs of selections that share a maintenance job.

The first term rewards parallel work; the second penalizes selecting overlapping pairs, preventing a job from being scheduled twice. Every valid selected pair saves one period, so minimizing energy corresponds to minimizing the number of maintenance periods.

With 10 jobs and two crews, the theoretical minimum is 5 periods.

I used a single-layer QAOA circuit to sample candidate solutions. An initial analytical scan of the QAOA parameter landscape on a 31 × 31 grid identified promising starting points, which were then refined through more focused optimization on the ideal Qiskit Aer simulator. The optimization emphasized the best feasible outcomes using a lower-tail CVaR objective, rather than simply minimizing average energy.

The trained circuit parameters were used for both noiseless Aer simulator sampling and for the IonQ Forte noisy simulator without retraining.

The idea of the hybrid approach is not to obtain an answer from a quantum circuit, but to generate a reasonably small pool of classically evaluated candidates, that contains good solution.

Finally, sampled solutions were filtered for feasibility and evaluated in batches of 16 accepted samples. The main performance measure was how often a group contained at least one optimal five-period schedule.

## 5. Results and Discussion

### Classical reference

For this small instance, classical enumeration is entirely sufficient to find the optimal schedules; there is no practical need for a classical or quantum optimizer.

The purpose of the experiment is instead to test whether QAOA can preferentially produce good solutions, using an exactly known classical reference for comparison.

Enumeration identified 1,216 feasible schedules, of which only 51 achieve the minimum of five maintenance periods. If 16 feasible schedules are sampled uniformly at random, the probability of finding at least one optimal solution in the batch of 16 samples is approximately 49.62%. This ~50% baseline defined my selection of the batch size.

### Ideal Qiskit Aer experiment

The trained QAOA circuit performed better than this reference in the ideal simulator experiment.

Out of 3.2 million raw samples, 32,216 passed the feasibility checks, giving an acceptance rate of approximately 1.00%.

Dividing these accepted samples into 2,013 complete groups of 16, 63.39% of groups contained an optimal five-period schedule. This represents an improvement of approximately 13.91 percentage points over uniform feasible sampling.

The observed result was also close to the 62.88% success probability predicted from the trained circuit's exact distribution.

The batch size provides a meaningful test of whether QAOA produces optimal schedules more reliably than uniform random sampling. Across 2,007 groups, the improvement was statistically significant under the independent-sampling binomial model: the observed 95% confidence interval of 61.24-65.50% lies clearly above the 49.62% reference.

The result supports the conclusion that the trained circuit preferentially samples optimal schedules, rather than behaving like a random sampler over feasible outcomes.

### IonQ Forte noisy-simulator experiment

With a feasibility rate of approximately 1%, I initially estimated that around 100,000 raw shots would be needed to obtain a representative sample and reduce statistical fluctuations. However, the IonQ Forte simulator has a practical execution limit: runs time out after approximately 60 minutes, limiting individual experiments to fewer than 15,000 shots. Therefore, I focus here on the most recent **12,000-shot execution**, with an earlier 10,000-shot run documented in the [GitHub repository](https://github.com/grigorovich-sergey/Qollab_IonQ_Hackathon_2026).

The noisy simulation reduced the feasibility rate by approximately half compared to ideal Aer, to just **0.48%**. This left relatively few accepted samples, making the results more suitable for qualitative analysis than for statistically reliable conclusions.

Nevertheless, the distribution of feasible schedule lengths shows an interesting shift. While uniformly sampled feasible schedules most frequently require seven periods, the IonQ noisy simulation produced a clear concentration of **six-period schedules**, suggesting a preference for shorter schedules even in the presence of noise.

Despite the limited sampling, **2 of the 3 complete batches of 16 feasible samples contained an optimal five-period schedule**, giving an observed success rate of **66.67%**. This is numerically comparable to the ideal Aer result (63.39%) and above the uniform-feasible reference (49.62%).

However, this observation should be interpreted with considerable caution. Out of 12,000 raw shots, only 58 feasible samples were obtained, allowing just three complete batches. The resulting **95% confidence interval is extremely wide (9.43–99.16%)**, making any conclusion about relative performance statistically unreliable. 

The simulator's execution timeout prevented collecting the much larger sample needed for a more representative analysis. The noisy experiment therefore demonstrates that optimal schedules can still be obtained, but does not establish an improvement over random sampling.


![Figure 3 — Comparison of feasible-schedule length distributions for uniform sampling, ideal Aer, and noisy IonQ.](https://pub.qollab.xyz/assets/01a12c6a-3a8a-7070-8906-8da568901665)
![Figure 4 — Probability of finding an optimal schedule in 16 accepted samples, including confidence intervals for the measured results.](https://pub.qollab.xyz/assets/01a12c6a-7249-79f2-9eef-6a5c1cfff7d3)


### Experimental Results Comparison

| Metric | Uniform feasible | Ideal Aer | IonQ Forte (noisy) |
|---|---:|---:|---:|
| Raw shots | — | 3,200,000 | 12,000 |
| Feasible samples | — | 32,216 | 58 |
| Acceptance rate | — | 1.01% | 0.48% |
| Optimal 5-period schedules among feasible samples | 4.19% | 5.92% | 5.1% |
| Complete groups of 16 samples | — | 2,013 | 3 |
| Group success rate | 49.62% (exact reference) | 63.39% | 66.67% (2 of 3) |
| 95% confidence interval | — | 61.24–65.50% | 9.43%–99.16% |
| Sampling runtime | — | 11.1 min | 53.7 min |

*Group success means finding at least one optimal five-period schedule among 16 feasible samples. The uniform reference is calculated exactly, while Aer and IonQ success rates are observed. IonQ groups were reconstructed from returned counts using a fixed shuffle seed.*

*The IonQ experiment produced only three complete groups, making its observed success rate statistically inconclusive. Its schedule-length distribution provides more useful qualitative evidence.*

### Selected schedules and limitations

Among the optimal schedules found in the ideal experiment, three shared the best cumulative route-travel-time score, approximately 69.06 minutes across five maintenance periods.

This secondary comparison helped distinguish equally short schedules without adding detour quality to the QAOA objective. The value is a sum of reference-trip durations over the maintenance periods, not a continuous journey or total population delay.

The experiment does not establish quantum computational advantage. The instance is small enough for exact classical enumeration, and all quantum circuits were simulated classically or with a quantum noise model.


## 6. Conclusion

This experiment demonstrates a **complete workflow** from real road-network open data to quantum-assisted maintenance scheduling. In the ideal simulation, the trained QAOA circuit produced optimal schedules more frequently than uniform random sampling of feasible solutions.

Although the problem is too small to require quantum optimization, it provides a useful test of a compact QUBO formulation and the behavior of QAOA under ideal and noisy simulation.

The noisy IonQ Forte experiment showed a shift toward shorter schedules and successfully produced optimal five-period solutions. However, the limited number of feasible samples and wide confidence interval prevented a reliable assessment of its performance relative to the ideal simulator or classical reference.

The experiment also illustrates **how real geographic constraints can be incorporated into quantum optimization problems**, even when the initial scheduling model is deliberately simplified.

On a personal experience, the most important observation is how much time, compared to my initial plan, was dedicated to QAOA optimization, parameter selection, starting seeds search strategy, feasibility control etc. This stage alone consumed nearly 15 hours of focused work. At the same time other tasks, such as geography data preprocessing and conversion, which I expected to be much longer, turned out to be fast and straightforward.

## Declaration of AI fair use and honor code

For this project, AI assistance was used for coding and code troubleshooting, text formatting and proofreading.

All decisions are owned by the author, whether related to experimental design, strategy, approach, solutions, workarounds and interpretation. All AI-written code was verified.

All parts of the presented project were created during the hackathon window. Parts of the work, performed in Google Colab notebooks, including geography data preprocessing, QAOA pretraining, and noiseless simulation sampling, were created within the same time window and committed to GitHub.

SHA sums of data source files are frozen in data_manifest.json and will remain unchanged to allow reproducibility experiments.

## References

1. Dixit, V. V., and Niu, C. (2023). *Quantum computing for transport network design problems.* Scientific Reports, 13, 12267. https://doi.org/10.1038/s41598-023-38787-2

2. Ozaeta, A., van Dam, W., and McMahon, P. L. (2022). *Expectation Values from the Single-Layer Quantum Approximate Optimization Algorithm on Ising Problems.* Quantum Science and Technology, 7, 045036. https://doi.org/10.1088/2058-9565/ac9013

**Data and software:** OpenStreetMap contributors, OSMnx, NetworkX, Qiskit, Qiskit Aer, Qollab and IonQ.

**Project links:** [GitHub repository](https://github.com/grigorovich-sergey/Qollab_IonQ_Hackathon_2026) [Qollab project](https://qollab.xyz/u/sergey_gris/road-closure-planning)

**This effort is supported by Qollab & IonQ.**
