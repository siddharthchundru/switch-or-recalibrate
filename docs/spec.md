# Design specification

**Working title:** Switching or Recalibrating? Two Kinds of Test-Time Adaptation in Robot Policies
**Repository:** `switch-or-recalibrate`
**Version:** 0.2 · October 2026

This document is the reference for implementation. Every number is a planning default or a hypothesis until the pilot fixes it. Hypotheses are labelled HB1–HB5 and mechanisms by short names.

---

## 1. Question and thesis

A pretrained robot policy is deployed, and something changes. There are two kinds of change.

- **World changes** (for example, a route becomes blocked). Sensory input still looks normal; what fails is the behaviour the policy relies on. Adapting means **switching** to a different behaviour.
- **Body changes** (for example, inverted or weakened actuators, or miscalibrated sensors). The relation between what the agent commands and what it then senses moves outside its trained range. Adapting means **recalibrating**.

**Thesis (a hypothesis).** These are different problems, solved by different mechanisms.
- Mechanisms that keep a use-dependent memory of the agent's own behaviour (habit-like) should switch well but not recalibrate.
- Mechanisms that restore internal or sensorimotor operating ranges (homeostasis-like) should recalibrate well but not switch.

**Enactive content.** Both mechanism families come from enactive and artificial-life work: habits as use-dependent sensorimotor patterns ([Egbert & Barandiaran, 2014](https://doi.org/10.3389/fnhum.2014.00590)), and homeostatic or ultrastable plasticity ([Di Paolo, 2000](https://sro.sussex.ac.uk/id/eprint/29695/), after Ashby). The paper is written for robot learning; this lineage appears in the introduction and the method sections.

**Beyond success rates.** For every mechanism we also characterise *how* the robot adapts (Section 8.3).

## 2. Position and contribution

The full literature check is in [`notes/stage0-literature-gate.md`](../notes/stage0-literature-gate.md).

**What prior work covers.** Test-time adaptation of robot policies is active and crowded:
- [TEMPO](https://arxiv.org/abs/2610.20318): gradient-based, critic-driven adaptation.
- [T²Mem](https://arxiv.org/abs/2609.36720) and [RoboTTT](https://research.nvidia.com/labs/gear/robottt/): fast-weight memories.
- [Retrieve-then-Steer](https://arxiv.org/abs/2605.10094) and [RESETTLE](https://arxiv.org/abs/2610.12185): gradient-free steering by retrieval.
- [Du et al., 2024](https://arxiv.org/abs/2406.15917): a stall trigger plus resampling away from past failures.

All of these report success after a shift.

**What none of them does, as far as the check found.**
1. Characterise the behavioural dynamics of adaptation.
2. Separate switching from recalibrating.

**Contributions, in priority order.**
1. **Track 2 (method): homeostatic test-time adaptation (HTA).** Gated, local plasticity in a frozen pretrained policy, with no reward, no task loss and no backpropagation. Novelty is provisional:
   - Gradient-free renormalisation exists for image classification.
   - Gradient-based normalisation-layer adaptation exists for RL ([Xu et al., 2025](https://proceedings.mlr.press/v267/xu25ar.html)).
   - Internal–behavioural coupling was studied in small evolved controllers (Iizuka & Di Paolo, 2008).
   - Stage 0b must pass before HTA is called new.
2. **Track 1 (evaluation).** A behavioural characterisation of switching mechanisms under world changes.
3. **Joint.** The mechanism × change-type interaction (HB5).

## 3. Hypotheses (registered per track before its main run)

- **HB1 (Track 2).** Under body changes, HTA reaches higher post-change success than Frozen and Renorm, without reward or gradients.
- **HB2 (Track 2).** Gating matters. Under no change, HTA−gate (plasticity always on) loses success relative to Frozen, while gated HTA does not.
- **HB3 (Track 1).** Among runs that switched, mechanisms trade switching latency against reversal after unblocking. Specific predictions:
  - FailAvoid switches fastest and returns least.
  - HabitMem's reversal improves with its decay rate, at a cost in latency.
  - Both are reported as trade-off curves over a pre-registered sweep (FailAvoid's record expiry; HabitMem's decay), not single points.
- **HB4 (Track 1).** For every switching mechanism, post-change success increases with the base policy's repertoire Rep (from `self-generated-experience`). This is tested by regressing on Rep with competence S_in as a covariate and base-policy seed as a random effect.
- **HB5 (joint).** In the 2 × 2 of {HabitMem, HTA} × {world change, body change}, the improvement over Frozen shows a crossover interaction: each mechanism helps more on its own change type. This is tested as a pre-registered interaction contrast on (mechanism − Frozen). With one mechanism per family, the claim is about these two mechanisms, not about whole families.

HB3 and HB5 are the most speculative. Clear nulls, shown by equivalence tests, count as answers.

## 4. Environments and embodiment ladder

| Rung | Environment | Use | Status |
| --- | --- | --- | --- |
| 1 | PointMaze M1 and M2 (shared with `self-generated-experience`); M0 for tuning | Both tracks; minimum viable experiments | MVE |
| 2 | A legged body (AntMaze or a MuJoCo locomotion task) | Generality | **Deferred and to be costed.** `self-generated-experience` does not produce legged base policies, and GCSL is weak on AntMaze. Base policies would need a separate source. |
| 3 | A public pretrained locomotion policy | Track 2 at realistic scale | Later |

The simulator version is frozen before `switch-or-recalibrate` starts. If `self-generated-experience` falls back to its NumPy simulator, `switch-or-recalibrate` uses the same one.

## 5. Base policies

`self-generated-experience`'s GCSL policies:
- C0, demonstrations only;
- C3, task self-practice (repertoire expected to collapse);
- C4, frontier novelty (repertoire expected to be preserved).

Each is frozen at deployment. Track 2 needs only C0, which exists early in `self-generated-experience`'s Stage 2. The planned order is `self-generated-experience`'s MVE first, then the Track 2 pilot. HB4 needs C3 and C4.

Seeds are nested: each mechanism runs several deployment seeds on each base-policy seed. The analysis treats the base-policy seed as a random effect, which avoids pseudo-replication.

## 6. Perturbations

| Class | Perturbation | Track | Role |
| --- | --- | --- | --- |
| World | Route block on the agent's most-used route (`self-generated-experience`'s P1); unblock after a fixed delay | 1, joint | Primary |
| World | Fixed block, the same for every policy (`self-generated-experience`'s P1-fixed) | 1 | Secondary; same maze for all |
| Body | **Observation miscalibration**: offset and gain on position | 2, joint | **Primary** body change (shifts the inputs directly) |
| Body | **Action inversion**: sign flip of one or both axes | 2, joint | Primary; needs the contingency monitor (Section 7.3) |
| Body | Action gain loss (×0.5–0.7, within the range a non-saturating policy can compensate for) | 2 | Secondary |
| Body | Added action noise | 2 | Robustness probe only. Recalibration cannot remove noise, so it supports no recalibration claim. |
| Control | No change | Both | Cost of adapting |

## 7. Deployment protocol and mechanisms

### 7.1 Protocol (identical for every mechanism)

- **Stream.** Deployment is one continuous stream of K episodes with start–goal pairs from a fixed seeded list. The change begins at a fixed episode and step and persists; for the reversal test it is undone at a fixed later point. Memory and plastic parameters carry across episodes.
- **Budget.** Every mechanism gets the same post-change environment steps B. FineTune uses the same gradient-steps-per-environment-step ratio as `self-generated-experience`'s training, with a fixed learning rate.
- **Tuning.** Every mechanism gets the same number of tuning trials, on M0 and on perturbation magnitudes not used in evaluation. All settings are then frozen.
- **Stall detector.** One shared, pre-registered detector, built from observations only. It fires when net displacement over the last W steps falls below δ, or when Euclidean distance to the goal has not decreased over W′ steps. It uses no map and no reward.
  - The detector is available to every mechanism.
  - Mechanisms that use it are marked in the tables.
  - kNN has a gated variant (kNN+gate) so that HabitMem's gating has a fair comparison.
- **Signals.** Each mechanism's table row lists exactly which signals it uses. No mechanism is told the perturbation type or the moment of change.

### 7.2 Track 1: switching mechanisms (world changes)

**MVE set**, capped to keep the study feasible: Frozen, FineTune, kNN, FailAvoid, HabitMem. FastWeight and SuccessSteer are added in Stage 3 only if the MVE is decided.

| Name | Mechanism | Signals used | Gradient-free |
| --- | --- | --- | --- |
| Frozen | No adaptation | — | — |
| FineTune | Online GCSL fine-tuning on post-change experience, hindsight-relabelled | Own trajectories, goals | No |
| kNN / kNN+gate | Non-parametric retrieval over the agent's own (state, goal, action) records, no plasticity; the gated variant retrieves only while stalled | Own records (+ stall) | Yes |
| FailAvoid | Adapted from Du et al. On stall, record the actions taken at the stall state. Resample policy candidates away from recorded actions in action space (no dynamics model). Stall detection uses a progress estimator trained on D0 (disclosed). Records expire after a swept horizon. | Stall, D0-trained progress estimator | Yes |
| HabitMem | Kernel-attention memory of the agent's own (state, goal, action) entries. Entries strengthen with use and decay with disuse; plasticity is gated by stall. | Own records, stall | Yes |
| HabitMem−R, −D, −G | HabitMem without reinforcement, without decay, without gating | As above | Yes |
| FastWeight (Stage 3) | Linear fast-weight layer updated by a self-supervised loss (small TTT analogue) | Own trajectories | No (local gradient) |
| SuccessSteer (Stage 3) | Bias sampling toward retrieved successful segments | Own records, success flag | Yes |

### 7.3 Track 2: recalibrating mechanisms (body changes)

**HTA (this work)** has two monitors, each paired with a plasticity rule.

- **Internal monitor and directed rule.**
  - Unit ranges [lo_j, hi_j] are recorded on rollouts of the frozen policy in the unperturbed environment (2.5–97.5% quantiles), not on D0, whose narrow range would trigger plasticity even without a change.
  - At deployment, each unit tracks a running average of its activation. Out-of-range units adjust their own gain g_j and bias b_j toward their range (intrinsic plasticity, after Triesch, 2005).
  - Output-layer logit gains and biases are included, because hidden-layer changes alone may not be able to alter the categorical action bins.
  - This rule targets observation miscalibration, which shifts the inputs directly.
- **Contingency monitor and ultrastable rule.**
  - A memoryless policy may not register an action inversion: its inputs stay in range. A separate monitor therefore compares the commanded action (an efference copy) with the observed change in velocity, and tracks the range of this command–outcome relation in the unperturbed environment.
  - While it is out of range, the policy's gains and biases take small random steps, and are kept once the relation returns to range (Ashby's ultrastability, Di Paolo, 2000).
  - The efference copy is *not* a policy input, which avoids the imitation-learning "copycat" problem.
  - This rule targets action inversion and gain loss.
- **Pre-pilot diagnostic, before any claim.** Measure the fraction of out-of-range units, and the contingency-monitor signal, under each perturbation with the Frozen policy. If a perturbation does not move a monitor, HTA cannot be expected to respond to it, and that pairing is dropped.
- **Internal–behavioural coupling, the open research problem.** The MVE uses no coupling objective. Stage 2 tests pretraining objectives that tie internal ranges to competence.

**Baselines and ablations:**

| Name | Mechanism | Uses source statistics | Gradient-free |
| --- | --- | --- | --- |
| Frozen | No adaptation | — | — |
| Renorm | Standard test-time renormalisation: re-standardise each unit's pre-activation with running test statistics, ungated | No | Yes |
| EntTTA | Entropy minimisation over gain and bias (after Xu et al., 2025) | No | No |
| FineTune | Online GCSL fine-tuning (upper reference) | — | No |
| HTA−gate | HTA with plasticity always on | Yes | Yes |
| HTA−global | One shared gain and bias instead of per-unit parameters | Yes | Yes |

HTA uses statistics from unperturbed rollouts; Renorm does not. This difference is disclosed. HTA−gate separates the effect of gating from the effect of having source statistics. Giving Renorm the source statistics would make it the same method as HTA−gate.

## 8. Measures

### 8.1 Success (comparable with prior work)

- Post-change success per episode.
- Area under the post-change success curve.
- Success under no change.

### 8.2 Adaptation dynamics (the battery, used as diagnostics)

| Measure | Definition | Track |
| --- | --- | --- |
| Perseveration (world) | Fraction of post-change steps spent inside the blocked corridor | 1 |
| Perseveration (body) | Fraction of stalled steps during which the adapted policy stays within a KL threshold of the frozen policy. Secondary, because under gain loss staying close to the old policy can be correct. | 2 |
| Switch or recovery latency | Steps from the change to the first success, as survival times censored at B | 1, 2 |
| Reversal | Among runs that switched only: probability of returning to the original shorter route within W steps of unblocking | 1 |
| Decay | Usage of the abandoned route over time since abandonment | 1 |
| Stability | Drift in behaviour and in success under no change | 1, 2 |
| Repertoire dependence | All of the above against base-policy Rep, with S_in as a covariate | 1 |

### 8.3 Behavioural characterisation: how the robot adapts

Each post-change trajectory window gets **two separate labels plus a success flag**. Windows are of equal length before and after the change.

1. **Route class.** North, south or neither, from the planner's cell sets (`src/envs/mazes.py`).
2. **Motor cluster.** Computed in *outcome space*: realised displacement, speed profile, heading-change and curvature statistics, and wall-contact rate.
   - Commanded actions are excluded. After an action inversion, any adapted policy would otherwise look new by definition.
   - Clusters are fit on pre-change windows. A window is "outside" the pre-change repertoire if its distance to the nearest cluster exceeds the 95th percentile of distances for held-out pre-change windows.
   - The clustering method and the number of clusters are fixed in the pre-registration, and sensitivity across cluster counts is reported.

**Derived categories.**

| Category | Route | Motor cluster | Example |
| --- | --- | --- | --- |
| Recovered | Old | Old | Back to the pre-change behaviour |
| Rerouted | New | Old | Takes the other arm with familiar movement |
| Compensated | Old | New | Same route, new way of moving |
| Novel | New | New | New route and new movement |

Each category is reported with and without success.

**Outputs.**
- Per-mechanism distributions over these categories.
- The number of new clusters and their distance from the pre-change repertoire.
- Hand-checked exemplar trajectories and videos.

**Status.** Descriptive and exploratory.

## 9. Analysis

- **Pre-registration on OSF** per track, before that track's main run.
- **Paired and nested design.** Within a deployment seed, all mechanisms share the base policy, the change onset and the noise streams.
- **Seeds.** 10-seed pilot followed by a power analysis; default 30 deployment seeds per cell.
- **Statistical methods.**
  - Latency: survival analysis (Kaplan–Meier curves, Cox models, restricted mean survival time).
  - Binary outcomes: logistic mixed models with base-policy seed as a random effect.
  - Holm correction within each track's confirmatory family.
  - Equivalence tests for null claims.

## 10. Plan and gates

| Stage | Work | Gate |
| --- | --- | --- |
| 0a | Literature gate, Track 1 ✔ | ✔ |
| 0b | Literature gate, Track 2: full texts of Iizuka & Di Paolo (2008), deep-RL normalisation TTA, intrinsic plasticity in deep networks, the dormant-neuron work | HTA not already published for control policies |
| 1 | **Track 2 pilot** on rung 1 with C0: pre-pilot monitor diagnostic; observation miscalibration and action inversion; HTA (both rules), Frozen, Renorm, EntTTA; 10 seeds | Some HTA variant recovers measurably more than Frozen and Renorm. Otherwise Track 2 is parked and Track 1 proceeds. |
| 2 | Track 2 main: coupling objectives, ablations, behavioural characterisation, pre-registered run | HB1 and HB2 decided |
| 3 | Track 1 MVE (Frozen, FineTune, kNN, FailAvoid, HabitMem + ablations) on C0, C3 and C4; sweeps; battery; characterisation | HB3 and HB4 decided |
| 4 | Joint 2 × 2 (HB5); FastWeight and SuccessSteer if warranted; rung 2 if costed; report | — |

**Order:** `self-generated-experience`'s MVE → 0b → 1, then 2 and 3 as the results warrant.

**Development process.** Each stage's implementation is reviewed by an independent auditing agent, which checks:
- equal post-change budgets;
- that "frozen" parameters really are frozen;
- the signals each mechanism uses, against Section 7;
- that no mechanism sees the perturbation type or the onset;
- that nothing claimed to be reward-free uses reward.

## 11. Risks

- **HTA may restore internal statistics without restoring behaviour** (the coupling problem). Mitigations: the pre-pilot diagnostic, the pilot gate, and Track 1 as a standalone result.
- **Small analogues may not represent large-model systems.** Mitigations: describe them honestly as analogues, and use the embodiment ladder once it is costed.
- **The field moves monthly.** Repeat the literature gates before each pre-registration.
- **Clustering can manufacture "novel" behaviour.** Mitigations: outcome-space descriptors, fixed thresholds, sensitivity analysis and hand checks.
- **Workload.** Capped MVE sets; rung 2 deferred.
