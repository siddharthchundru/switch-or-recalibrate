# Design specification

**Working title:** Switching, Recalibrating or Relearning? Kinds of Test-Time Adaptation in Robot Policies
**Repository:** `switch-or-recalibrate`
**Version:** 0.3 · 26 October 2026 · supersedes v0.2 (23 October 2026)

This document is the reference for implementation. Every number is a planning default or a hypothesis until the pilot fixes it. Hypotheses are labelled HB1–HB7, and mechanisms by short names, to avoid clashes with the labels in [self-generated-experience](https://github.com/siddharthchundru/self-generated-experience) ("Project A" below).

**Changes in v0.3, in brief:**
1. Three kinds of change instead of two. Action inversion moves from "recalibrate" to a new class, "relearn".
2. Mechanisms are grouped into three families (habit, homeostatic, error-based) with at least two members each, so claims can be made about families.
3. A screening stage selects which mechanisms go into the confirmatory run.
4. Aftereffects and savings are added to the behavioural battery.
5. The contribution is narrowed after a preliminary Track 2 literature check, and new baselines are added.

---

## 1. Question and thesis

A pretrained robot policy is deployed, and something changes. There are three kinds of change.

| Kind | Examples | What adapting means |
| --- | --- | --- |
| **World change** | A route becomes blocked | **Switching** to a different behaviour the policy already has. Sensory input still looks normal; what fails is the behaviour. |
| **Graded body change** | Weaker actuators (gain loss); a miscalibrated position sensor (offset or gain) | **Recalibrating**: keeping the same behaviour, adjusted. The relation between command and outcome shifts but keeps its sign and structure. |
| **Reversal** | Action inversion (the sign of one or both axes flipped) | **Relearning**: building a new controller. The relation between command and outcome changes direction. |

**Why reversal is separate.** In human motor learning, gains, offsets and rotations are handled by error-based adaptation: fast, and leaving aftereffects when the perturbation is removed. Mirror reversal is learned de novo instead: slower, with no aftereffects, and error-based adaptation can even act in the wrong direction (Yang, Cowan & Haith, eLife 2021; Telgen et al., J. Neurosci 2014). Treating inversion as recalibration would merge two different problems.

**Thesis (a hypothesis).** Different kinds of change are solved by different families of mechanism.
- **Habit family** (use-dependent memory of the agent's own behaviour): switches well; does not recalibrate or relearn.
- **Homeostatic family** (restore internal or sensorimotor operating ranges): recalibrates without an error model. The ultrastable member can, in principle, relearn, slowly.
- **Error-based family** (correct from the error between predicted and observed outcome): recalibrates graded changes well, with aftereffects and savings; fails, or pushes the wrong way, under reversal.

**Lineage.** The habit and homeostatic families come from enactive and artificial-life work: habits as use-dependent sensorimotor patterns ([Egbert & Barandiaran, 2014](https://doi.org/10.3389/fnhum.2014.00590)), and homeostatic or ultrastable plasticity ([Di Paolo, 2000](https://sro.sussex.ac.uk/id/eprint/29695/), after Ashby). The error-based family comes from motor neuroscience (Smith, Ghazizadeh & Shadmehr, 2006). The paper is written for robot learning; this lineage appears in the introduction and the method sections.

**Beyond success rates.** For every mechanism we also characterise *how* the robot adapts (Section 8.3), including the aftereffect and savings signatures that separate recalibration from relearning (Section 8.2).

## 2. Position and contribution

The Track 1 literature check is in [`notes/stage0-literature-gate.md`](../notes/stage0-literature-gate.md). The Track 2 check has had a preliminary pass (from abstracts and paper pages); its memo will be added to `notes/` when the full check is done.

**What prior work covers.** Test-time adaptation of robot policies is active and crowded, and four of the closest papers appeared in September and October 2026.
- Gradient-based, critic-driven adaptation to body and dynamics changes: [TEMPO](https://arxiv.org/abs/2610.20318).
- Gradient fine-tuning of a frozen vision-language-action model from the command–outcome residual, with no reward: [self-compensation, Lee et al., 2026](https://arxiv.org/abs/2609.37334).
- Action–outcome mismatch updating a meta-trained belief latent: [SCOUT](https://arxiv.org/abs/2609.36107).
- A learned adapter driven by motor telemetry: [TeAR](https://arxiv.org/abs/2609.36182).
- Fast-weight memories: [T²Mem](https://arxiv.org/abs/2609.36720), [RoboTTT](https://research.nvidia.com/labs/gear/robottt/).
- Gradient-free steering by retrieval: [Retrieve-then-Steer](https://arxiv.org/abs/2605.10094), [RESETTLE](https://arxiv.org/abs/2610.12185).
- A stall trigger plus resampling away from past failures: [Du et al., 2024](https://arxiv.org/abs/2406.15917).
- Gradient-free, forward-only adaptation that aligns activation statistics, in image classification: [FOA](https://arxiv.org/abs/2404.01650).

**Not claimed as new:**
- Reward-free adaptation of frozen robot policies to body changes.
- Using the mismatch between commanded and observed motion as the adaptation signal. HTA's contingency monitor follows Lee et al. and SCOUT.

**What none of them does, as far as the checks found.**
1. Adapt a frozen pretrained control policy with **no gradients, no learned module and no meta-training**, using local homeostatic or ultrastable rules.
2. Characterise the behavioural dynamics of adaptation.
3. Separate switching, recalibrating and relearning.

**Contributions, in priority order.**
1. **Track 2 (method): homeostatic test-time adaptation (HTA).** Gated, gradient-free, training-free plasticity in a frozen pretrained policy. Novelty is provisional until the full Track 2 literature check passes. HTA has to be shown competitive with gradient methods on at least some body changes. Otherwise it is framed honestly as a cheap first response that comes before gradient adaptation.
2. **Track 1 (evaluation).** A behavioural characterisation of switching mechanisms under world changes.
3. **Joint.** The mechanism-family × change-type interaction (HB5), and the recalibrate-versus-relearn dissociation (HB6, HB7).

## 3. Hypotheses (registered per track before its main run)

- **HB1 (Track 2).** Under graded body changes, HTA reaches higher post-change success than Frozen and Renorm, without reward or gradients.
- **HB2 (Track 2).** Gating matters. Under no change, HTA−gate (plasticity always on) loses success relative to Frozen, while gated HTA does not.
- **HB3 (Track 1).** Among runs that switched, mechanisms trade switching latency against reversal after unblocking. Specific predictions:
  - FailAvoid switches fastest and returns least.
  - HabitMem's reversal improves with its decay rate, at a cost in latency.
  - Both are reported as trade-off curves over a pre-registered sweep (FailAvoid's record expiry; HabitMem's decay), not single points.
- **HB4 (Track 1).** For every switching mechanism, post-change success increases with the base policy's repertoire Rep (from Project A). This is tested by regressing on Rep with competence S_in as a covariate and base-policy seed as a random effect.
- **HB5 (joint).** In the families × {world change, graded body change} design, the improvement over Frozen shows a crossover interaction: each family helps more on its own change type. This is tested as a pre-registered interaction contrast on (mechanism − Frozen), with mechanism nested in family. With two selected mechanisms per family, the claim is about these families as sampled, not about every possible member.
- **HB6 (joint).** Under reversal (action inversion), error-based adaptation and HTA's directed rule do not improve on Frozen, and may do worse; the Homeostat improves on Frozen, more slowly than under graded changes.
- **HB7 (joint).** When a graded body change is undone, error-based adaptation and HTA show aftereffects (errors in the opposite direction) and savings on re-exposure. When the reversal is undone, no mechanism that relearned shows a comparable aftereffect.

HB3, HB5 and HB6 are the most speculative. Clear nulls, shown by equivalence tests, count as answers.

## 4. Environments and embodiment ladder

| Rung | Environment | Use | Status |
| --- | --- | --- | --- |
| 1 | PointMaze M1 and M2 (shared with Project A); M0 for tuning | Both tracks; minimum viable experiments | Minimum viable experiment |
| 2 | A legged body (AntMaze or a MuJoCo locomotion task) | Generality | **Deferred and to be costed.** Project A does not produce legged base policies, and GCSL is weak on AntMaze. Base policies would need a separate source. |
| 3 | A public pretrained locomotion policy | Track 2 at realistic scale | Later. Competing methods are tested on real legged robots and large policies, so rung 2 or 3 is needed for a main-venue paper. |

The simulator version is frozen before this project starts. If Project A falls back to its NumPy simulator, this project uses the same one.

**Known limitation of rung 1.** In a 2-D point mass, the body is nearly linear and fully identifiable, so the adaptive inverse (Section 7.4) will probably solve graded body changes and reversal almost perfectly. It is reported as a ceiling, not hidden. HTA's case rests on settings where the body cannot be identified explicitly, which is why rungs 2 and 3 matter.

## 5. Base policies

Project A's goal-conditioned supervised learning (GCSL) policies:
- demonstrations only;
- task-directed self-practice (repertoire expected to collapse);
- frontier-novelty practice (repertoire expected to be preserved).

Each is frozen at deployment. Track 2 needs only the demonstrations-only policy, which exists early in Project A's Stage 2. HB4 needs the two self-practice policies.

Seeds are nested: each mechanism runs several deployment seeds on each base-policy seed. The analysis treats the base-policy seed as a random effect, which avoids pseudo-replication.

## 6. Perturbations

| Class | Perturbation | Track | Role |
| --- | --- | --- | --- |
| World | Route block on the agent's most-used route (Project A's P1); unblock after a fixed delay | 1, joint | Primary |
| World | Fixed block, the same for every policy (Project A's P1-fixed) | 1 | Secondary; same maze for all |
| Graded body | **Observation miscalibration**: offset and gain on position | 2, joint | **Primary.** The offset is invisible to any displacement-based error, since a constant offset doesn't change velocity; only the gain is visible. This is a predicted dissociation between HTA and error-based adaptation. |
| Graded body | **Action gain loss** (×0.5–0.7, within the range a non-saturating policy can compensate for) | 2, joint | **Primary** |
| Reversal | **Action inversion**: sign flip of one or both axes | 2, joint | **Primary** for HB6 and HB7. Needs the contingency monitor (Section 7.3). |
| Body | Added action noise | 2 | Robustness probe only. Recalibration cannot remove noise, so it supports no recalibration claim. |
| Control | No change | All | Cost of adapting |

**Removal and re-exposure (for HB7).** In every body-change stream, the change is undone at a fixed point (to measure aftereffects), then re-applied later (to measure savings). Timings are fixed in the pre-registration.

## 7. Deployment protocol and mechanisms

### 7.1 Protocol (identical for every mechanism)

- **Stream.** Deployment is one continuous stream of K episodes with start–goal pairs from a fixed seeded list. The change begins at a fixed episode and step and persists. It is undone, and for body changes re-applied, at fixed later points (Section 6). Memory and plastic parameters carry across episodes.
- **Budget.** Every mechanism gets the same post-change environment steps B. Gradient-based mechanisms use the same gradient-steps-per-environment-step ratio as Project A's training, with a fixed learning rate.
- **Tuning.** Every mechanism gets the same number of tuning trials, on M0 and on perturbation magnitudes not used in evaluation. All settings are then frozen.
- **Stall detector.** One shared, pre-registered detector, built from observations only. It fires when net displacement over the last W steps falls below δ, or when Euclidean distance to the goal has not decreased over W′ steps. It uses no map and no reward.
  - The detector is available to every mechanism.
  - Mechanisms that use it are marked in the tables.
  - kNN has a gated variant (kNN+gate) so that HabitMem's gating has a fair comparison.
- **Forward model.** One shared, pre-registered linear model of the unperturbed body, fitted on Frozen-policy rollouts before deployment: the expected velocity change given the commanded action. It provides the predicted outcome for the contingency monitor, the error-based family and the adaptive inverse. Every mechanism that uses it is marked.
- **Monitor thresholds.** Out-of-range thresholds for every monitor are set from unperturbed rollouts, using prediction-error thresholding in the style of active-inference fault detection (Baioumy et al., 2021). They are fixed before evaluation.
- **Signals.** Each mechanism's table row lists exactly which signals it uses. No mechanism is told the perturbation type or the moment of change.

### 7.2 Habit family and Track 1 switching mechanisms (world changes)

| Name | Mechanism | Signals used | Gradient-free | Stage |
| --- | --- | --- | --- | --- |
| Frozen | No adaptation | — | — | Reference |
| FineTune | Online GCSL fine-tuning on post-change experience, hindsight-relabelled | Own trajectories, goals | No | Reference |
| kNN / kNN+gate | Non-parametric retrieval over the agent's own (state, goal, action) records, no plasticity; the gated variant retrieves only while stalled | Own records (+ stall) | Yes | Track 1 |
| FailAvoid | Adapted from Du et al. On stall, record the actions taken at the stall state. Resample policy candidates away from recorded actions in action space (no dynamics model). Stall detection uses a progress estimator trained on the demonstrations (disclosed). Records expire after a swept horizon. Related: T-resilience (Koos, Cully & Mouret, 2013). | Stall, demonstration-trained progress estimator | Yes | Track 1 |
| **HabitMem** (habit) | Kernel-attention memory of the agent's own (state, goal, action) entries. Entries strengthen with use and decay with disuse; plasticity is gated by stall. | Own records, stall | Yes | Screen |
| **IDSM residual** (habit) | A faithful Iterant Deformable Sensorimotor Medium (Egbert & Barandiaran, 2014) layered on the frozen policy. It records sensorimotor nodes along the trajectory, reinforces them by revisiting, lets them decay, and adds a residual motor term. Predicted to re-enact, not recalibrate, under body changes. | Own trajectory | Yes | Screen |
| HabitMem−R, −D, −G | HabitMem without reinforcement, without decay, without gating | As above | Yes | Track 1 ablations |
| FastWeight | Linear fast-weight layer updated by a self-supervised or Hebbian rule (small test-time-training analogue) | Own trajectories | No (local gradient) or yes (Hebbian) | Stage 4 |
| SuccessSteer | Bias sampling toward retrieved successful segments | Own records, success flag | Yes | Stage 4 |
| VISSA hybrid | IDSM extended with a local (1+1) evolution strategy that keeps sensorimotor patterns viable (Egbert, 2022): a habit–homeostat hybrid | Own trajectory, viability score | Yes | Optional, later |

### 7.3 Homeostatic family (body changes)

| Name | Mechanism | Signals used | Gradient-free | Stage |
| --- | --- | --- | --- | --- |
| **HTA, directed rule** | **Internal monitor.** Unit ranges [lo_j, hi_j] are recorded on unperturbed rollouts of the frozen policy (2.5–97.5% quantiles), not on the demonstrations, whose narrow range would trigger plasticity without any change. At deployment, each unit tracks a running average of its activation. Out-of-range units adjust their own gain g_j and bias b_j toward their range (intrinsic plasticity, after Triesch, 2005). Output-layer logit gains and biases are included, because hidden-layer changes alone may not move the categorical action bins. | Unit activations, source ranges | Yes | Screen |
| **Homeostat** | **Contingency monitor.** It compares the commanded action (an efference copy) with the observed velocity change via the shared forward model, and tracks the range of this relation in unperturbed rollouts. While it is out of range, gains and biases take small random steps, and are kept once the relation returns to range (Ashby's ultrastability; Di Paolo, 2000, which tested visual inversion). The efference copy is *not* a policy input, which avoids the imitation-learning "copycat" problem. | Command–outcome relation, forward model | Yes | Screen |
| **HTA (combined)** | The directed rule and the Homeostat together | Both | Yes | Screen |
| **Homeostatic synaptic plasticity** | One small plastic layer inserted into the frozen policy (initialised to identity). Its weights change by a Hebbian-type rule only while its units are out of range (Di Paolo, 2000; Iizuka & Di Paolo, 2007; Barandiaran & Di Paolo, 2010). | Unit activations, source ranges | Yes | Screen |
| HTA−gate | HTA with plasticity always on | As HTA | Yes | Ablation |
| HTA−global | One shared gain and bias instead of per-unit parameters | As HTA | Yes | Ablation |

**Pre-pilot diagnostic, before any claim.** With the Frozen policy, measure the fraction of out-of-range units and the contingency-monitor signal under each perturbation. If a perturbation does not move a monitor, a mechanism using that monitor cannot be expected to respond to it, and that pairing is dropped.

**Internal–behavioural coupling, the open research problem.** HTA may restore internal statistics without restoring behaviour. The minimum viable experiment uses no coupling objective. Stage 2 tests pretraining objectives that tie internal ranges to competence.

### 7.4 Error-based family and references

| Name | Mechanism | Signals used | Gradient-free | Stage |
| --- | --- | --- | --- | --- |
| **Two-state adaptation** (error-based) | A correction at the action interface, a = a_policy ⊙ g + c, with per-axis gain g and offset c. Each is the sum of a fast state and a slow state, x_f and x_s. Each state updates as x ← A·x + B·e, where e is the prediction error from the shared forward model, A is retention (fast: lower; slow: near 1) and B is the learning rate (fast: higher; slow: lower) (Smith, Ghazizadeh & Shadmehr, 2006). | Command–outcome error, forward model | Yes | Screen |
| Two-state, single rate | One state instead of two (ablation; tests whether the fast/slow split matters for savings) | As above | Yes | Ablation |
| **Adaptive inverse** (reference, ceiling) | Fits the full 2 × 2 map from commanded action to realised velocity online by recursive least squares, and corrects actions through its inverse. It can represent inversion; the two-state model cannot. | Command–outcome pairs | Yes | Reference |
| Renorm | Standard test-time renormalisation: re-standardise each unit's pre-activation with running test statistics, ungated | Unit activations | Yes | Reference |
| EntTTA | Entropy minimisation over gain and bias (after Xu et al., 2025) | Policy outputs | No | Reference |
| **Residual fine-tune** | Small analogue of Lee et al. (2026): gradient fine-tuning toward the command–outcome pseudo-target | Command–outcome residual | No | Reference |
| **Forward-only alignment** | Analogue of FOA (Niu et al., 2024): CMA-ES on gains and biases to match source activation statistics | Unit activations, source statistics | Yes | Reference |

HTA uses statistics from unperturbed rollouts; Renorm does not. This difference is disclosed. HTA−gate separates the effect of gating from the effect of having source statistics. Giving Renorm the source statistics would make it the same method as HTA−gate.

**Predicted dissociations, written down now:**

| Change | Two-state adaptation | HTA, directed rule | Homeostat | Adaptive inverse |
| --- | --- | --- | --- | --- |
| Action gain loss | Fixes it; aftereffect on removal | Partial | Slow | Fixes it |
| Observation offset | Blind to it (no displacement error) | Responds (inputs shift) | Blind to it (no contingency change) | Blind to it |
| Observation gain | Fixes it | Responds | Slow | Fixes it |
| Action inversion | Pushes the wrong way | Little or none | Slow, possible | Fixes it |

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
| Switch or recovery latency | Steps from the change to the first success, as survival times censored at B | All |
| Reversal | Among runs that switched only: probability of returning to the original shorter route within W steps of unblocking | 1 |
| Decay | Usage of the abandoned route over time since abandonment | 1 |
| **Aftereffect** | After a body change is undone: the signed error between realised and intended movement over the first N steps, relative to the pre-change baseline. A significant opposite-signed error indicates recalibration. | 2, joint |
| **Savings** | Ratio of recovery latency on the second exposure to the same change to latency on the first | 2, joint |
| Stability | Drift in behaviour and in success under no change | All |
| Repertoire dependence | All of the above against base-policy Rep, with S_in as a covariate | 1 |

### 8.3 Behavioural characterisation: how the robot adapts

Each post-change trajectory window gets **two separate labels plus a success flag**. Windows are of equal length before and after the change.

1. **Route class.** North, south or neither, from the planner's cell sets (Project A's `src/envs/mazes.py`).
2. **Motor cluster.** Computed in *outcome space*: realised displacement, speed profile, heading-change and curvature statistics, and wall-contact rate.
   - Commanded actions are excluded. After an action inversion, any adapted policy would otherwise look new by definition.
   - Clusters are fit on pre-change windows. A window is "outside" the pre-change repertoire if its distance to the nearest cluster exceeds the 95th percentile of distances for held-out pre-change windows.
   - The clustering method and the number of clusters are fixed in the pre-registration, and sensitivity across cluster counts is reported.
   - Descriptor choice is informed by computational sensorimotor-contingency work (Laflaquière et al.).

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

- **Screening, then confirmation.**
  - **Screening pilot:** every mechanism marked "Screen" plus all references; 10 seeds; route block, observation miscalibration, action gain loss, action inversion and no change.
  - **Selection rule, fixed before the screen:** per family, the two mechanisms with the highest area under the post-change success curve on that family's predicted change type, with stability under no change as the tiebreak. All references are carried forward.
  - Screening results are reported in full and labelled exploratory. Confirmatory claims cover only the selected mechanisms.
- **Pre-registration on OSF** per track, after the screen and before that track's main run.
- **Paired and nested design.** Within a deployment seed, all mechanisms share the base policy, the change onset and the noise streams.
- **Seeds.** Power analysis from the screening pilot; default 30 deployment seeds per cell.
- **Statistical methods.**
  - Latency: survival analysis (Kaplan–Meier curves, Cox models, restricted mean survival time).
  - Binary outcomes: logistic mixed models with base-policy seed as a random effect.
  - HB5: interaction contrast with mechanism nested in family.
  - Aftereffects: signed-error tests against the pre-change baseline.
  - Holm correction within each track's confirmatory family.
  - Equivalence tests for null claims.

## 10. Plan and gates

| Stage | Work | Gate |
| --- | --- | --- |
| 0a | Literature check, Track 1 ✔ | ✔ |
| 0b | Literature check, Track 2. Preliminary pass done 26 October 2026. Remaining: full texts of Iizuka & Di Paolo (2008), Lee et al. (2026), SCOUT and FOA; plasticity applied to pretrained deep policies (2023–26); GRACE (arXiv 2607.21661); re-check Track 1 against September–October 2026 papers. | HTA not already published for control policies |
| 1 | **Build and screen** on rung 1 with the demonstrations-only base policy: shared forward model; pre-pilot monitor diagnostic; every "Screen" mechanism plus references; 10 seeds | Some homeostatic mechanism recovers measurably more than Frozen and Renorm on a graded body change. Otherwise Track 2 is parked and Track 1 proceeds. |
| 2 | Track 2 main: selected mechanisms, coupling objectives, ablations, aftereffects and savings, behavioural characterisation, pre-registered run | HB1, HB2, HB6, HB7 decided |
| 3 | Track 1 (Frozen, FineTune, kNN, FailAvoid, selected habit mechanisms + ablations) on all three base policies; sweeps; battery; characterisation | HB3 and HB4 decided |
| 4 | Joint family × change-type design (HB5); FastWeight, SuccessSteer and VISSA if warranted; rung 2 if costed; report | — |

**Order:** Project A's minimum viable experiment → 0b → 1, then 2 and 3 as the results warrant.

**Development process.** One agent implements and a separate agent audits each stage. Audit reports are committed to `notes/audits/`. The audit checks:
- equal post-change budgets;
- that "frozen" parameters really are frozen;
- the signals each mechanism uses, against Section 7;
- that no mechanism sees the perturbation type or the onset;
- that nothing claimed to be reward-free uses reward;
- that the screening selection rule was applied as written.

## 11. Risks

- **HTA may restore internal statistics without restoring behaviour** (the coupling problem). Mitigations: the pre-pilot diagnostic, the screening gate, and Track 1 as a standalone result.
- **The adaptive inverse may dominate on rung 1.** Expected and reported. The claim about HTA rests on the family dissociations and on rungs 2–3, not on beating an explicit model in a 2-D body.
- **More mechanisms mean more forking paths.** Mitigations: selection rule fixed before the screen, screening labelled exploratory, Holm correction.
- **Small analogues may not represent large-model systems.** Mitigations: describe them honestly as analogues, and use the embodiment ladder once it is costed.
- **The field moves monthly.** Repeat the literature checks before each pre-registration.
- **Clustering can manufacture "novel" behaviour.** Mitigations: outcome-space descriptors, fixed thresholds, sensitivity analysis and hand checks.
- **Workload.** Screening keeps the confirmatory set capped; rung 2 is deferred.
