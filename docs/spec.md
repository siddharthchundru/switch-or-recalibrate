# Design specification

**Working title:** Switching, Recalibrating or Relearning? Kinds of Test-Time Adaptation in Robot Policies
**Repository:** `switch-or-recalibrate`
**Version:** 0.4.1 · 27 October 2026 · supersedes v0.4 (26 October 2026)

This document is the reference for implementation. Every number is a planning default or a hypothesis until the pilot fixes it. Hypotheses are labelled HB1–HB7, and mechanisms by short names, to avoid clashes with the labels in [self-generated-experience](https://github.com/siddharthchundru/self-generated-experience) ("Project A" below).

---

## 0. Changes in v0.4.1 and v0.4, and the reasons for them

**v0.4.1 (27 October 2026)** positions the project against repertoire-based damage recovery, after a literature check and an independent audit.
1. **Prior work.** Intelligent Trial and Error (Cully et al., 2015), Reset-free Trial and Error (Chatzilygeroudis et al., 2018) and adaptive prior selection (Kaushik et al., 2020) are added to Section 2. They are gradient-free but search an explicit repertoire using a task-performance signal, so the novelty claims as worded stand. T-resilience (Koos, Cully & Mouret, 2013) is credited as an early use of predicted-versus-observed mismatch for damage recovery.
2. **The thesis table is scoped to the point mass.** Intelligent Trial and Error recovers from body damage by *switching* to another stored behaviour. In a 2-D point mass there is no other way to move, so "graded body change → recalibrating" holds only at rung 1 (Section 1, Section 4).
3. **A repertoire-search reference.** "Map-guided trial and error", a small analogue of Intelligent Trial and Error over the base policy's own behaviours, is added to Track 1 in Stage 3 and is required before rung 2 (Section 7.2, Section 10).
4. **Store-and-select's lineage** now includes adaptive prior selection as a robotics precedent (Section 7.5).

**v0.4 (26 October 2026)**

v0.4 follows an external review of v0.3. The review found that the link between some update rules, the signals that drive them and the behaviour they are meant to restore was underspecified, and in one case inconsistent. Each change below says what changed and why.

1. **Each monitor now drives only parameters that can change what it monitors.** In v0.3 the Homeostat's contingency monitor (commanded action against observed motion) drove gains and biases *inside* the policy. Those parameters change what is commanded, but not how the body turns a command into motion, so the monitor could only return to range if the robot moved less. The Homeostat's parameters now sit at the **action interface**, between the policy and the body (Section 7.1). The general rule, after Ashby: a monitor may drive only parameters that lie inside the loop it monitors. Input-side monitors drive input-side parameters; the contingency monitor drives the action interface.
2. **The command copy is defined.** The command the monitors and error-based mechanisms read is the policy's own command, taken *before* any interface correction. "Intended movement" is the forward model applied to that command (Section 7.1).
3. **The directed rule's scope is stated.** Restoring activation statistics has a direct mechanism for observation changes (an input offset can be undone exactly by a bias change), but not for action changes, which leave the inputs unchanged. Predictions are revised accordingly (Section 7.3, Section 7.6), and the assumption that makes an observation offset recoverable is written down.
4. **Recalibrating and relearning are defined by behavioural signature, not by the kind of perturbation.** A mechanism that **overwrites** its parameters should show an aftereffect when the change is undone. A mechanism that **stores a new controller alongside the old one and selects between them** should show none, and should re-adapt quickly on re-exposure. A new mechanism, **Store-and-select**, tests the second pattern (Section 7.5). The Homeostat is now predicted to overwrite, so to show an aftereffect after reversal (HB7).
5. **The error-based family's limit under reversal is dynamic, not representational.** Its interface gains can become negative; what fails is the update rule, which assumes the unperturbed body's sign and so pushes the wrong way under inversion (Section 7.4).
6. **The error-based family has a second member.** The residual fine-tune moves from the references into the family; like the two-state model, its correction assumes the unperturbed body's sign. The adaptive inverse stays a reference, because it identifies the body's sign rather than assuming it.
7. **The habit family's prediction is revised.** Use-dependent strengthening at a stall strengthens the stalled behaviour, so habit mechanisms are now predicted to **persist** with a blocked route, switch slowly if at all, and return readily once it reopens. HabitMem's stall signal now controls when retrieval affects action, not when plasticity happens. The viability-based habit–homeostat hybrid (VISSA) moves into the screen (Section 7.2). HB3 and HB5 are revised.
8. **The screening selection rule no longer favours the main hypothesis.** v0.3 picked each family's best mechanisms on that family's *predicted* change type. v0.4 applies a viability gate, then ranks by mean performance across all change types, and fixes theoretical criteria for family membership before the screen (Section 9).
9. **The novelty claim is reworded.** The method uses no deployment-time gradients and no meta-training, but some variants use one linear forward model fitted before deployment. The claim now says so (Section 2).
10. **The stall detector drops its goal-distance clause.** Distance to the goal is essentially PointMaze's dense reward and is computed from observations that may be miscalibrated. The detector now uses net displacement only (Section 7.1).
11. **New controls and diagnostics.** Two mechanistic checks before the screen; detection, restoration and recovery reported separately; false alarms under world changes; a reference for movement changes imposed by the body rather than learned; computational cost; a direct manipulation of repertoire for memory-based mechanisms; pretraining crossed with mechanism in Stage 2.
12. **A scope tripwire** (Section 10).

---

## 1. Question and thesis

A pretrained robot policy is deployed, and something changes. There are three kinds of change.

| Kind of change | Examples | What adapting is expected to require |
| --- | --- | --- |
| **World change** | A route becomes blocked | **Switching** to a different behaviour the policy already has. Sensory input still looks normal; what fails is the behaviour. |
| **Graded body change** | Weaker actuators (gain loss); a miscalibrated position sensor (offset or gain) | **Recalibrating**: keeping the same behaviour, adjusted. The relation between command and outcome shifts but keeps its sign and structure. |
| **Reversal** | Action inversion (the sign of one or both axes flipped) | **Relearning**: acquiring a new controller. The relation between command and outcome changes direction. |

The first column describes the change. Whether a mechanism actually recalibrated or relearned is judged by what it does, using behavioural signatures (below), not assumed from the perturbation.

**The mapping from change to adaptation is specific to the point mass.** A body with several ways of moving can meet damage by switching to another stored behaviour: Intelligent Trial and Error recovers a damaged hexapod by searching a repertoire of gaits (Cully et al., 2015). In a 2-D point mass there is no alternative way to move, so a body change can only be met by recalibrating or relearning. At rung 2 the table becomes a hypothesis about which kind of adaptation each mechanism uses, not a property of the change (Section 4).

**Why reversal is separate.** In human motor learning, gains, offsets and rotations are handled by error-based adaptation: fast, and leaving aftereffects when the perturbation is removed. Mirror reversal is learned de novo instead: slower, with no aftereffects, and error-based adaptation can even act in the wrong direction (Yang, Cowan & Haith, eLife 2021; Telgen et al., J. Neurosci 2014). Treating inversion as recalibration would merge two different problems.

**Operational definitions (new in v0.4).**
- **Recalibration** is adaptation that changes the existing controller in place. Signature: an aftereffect (error in the opposite direction) when the change is undone, and at most partial savings on re-exposure.
- **Relearning**, in the sense tested here, is adaptation that acquires a new controller while keeping the old one. Signature: no aftereffect beyond the time needed to detect the change, and fast re-adaptation (savings) on re-exposure, because the stored controller is selected again.

A mechanism that recovers from reversal by overwriting its parameters has, by these definitions, recalibrated through a large change, not relearned. One interpretation of the de novo learning results, which this project tests rather than assumes, is that relearning requires storing and selecting between controllers (Wolpert & Kawato, 1998; Heald, Lengyel & Wolpert, 2021).

**Thesis (a hypothesis).** Different kinds of change are handled by different families of mechanism, and the families leave different behavioural signatures.
- **Habit family** (use-dependent memory of the agent's own behaviour): persists with established behaviour. Predicted to switch slowly, if at all, at a rate set by decay, and to return readily once the original route reopens. Does not recalibrate or relearn.
- **Homeostatic family** (restore internal or sensorimotor operating ranges): recalibrates without an error model, each rule within the loop its monitor covers. The ultrastable member can, in principle, recover from reversal, slowly, by overwriting.
- **Error-based family** (correct from the error between intended and realised outcome, assuming the unperturbed body's sign): recalibrates graded changes well, with aftereffects and savings; fails, or pushes the wrong way, under reversal.
- **Store-and-select** (a hybrid): recovers from reversal like the ultrastable member, but without an aftereffect, and with fast re-adaptation on re-exposure.

**Lineage.** The habit and homeostatic families come from enactive and artificial-life work: habits as use-dependent sensorimotor patterns ([Egbert & Barandiaran, 2014](https://doi.org/10.3389/fnhum.2014.00590)), and homeostatic or ultrastable plasticity ([Di Paolo, 2000](https://sro.sussex.ac.uk/id/eprint/29695/), after Ashby). The error-based family comes from motor neuroscience (Smith, Ghazizadeh & Shadmehr, 2006). Store-and-select follows the modular and contextual-inference accounts of motor memory (Wolpert & Kawato, 1998; Heald, Lengyel & Wolpert, 2021). That habitual behaviour persists after its outcome stops paying off is a classic finding in animal learning (Adams & Dickinson, 1981). The paper is written for robot learning; this lineage appears in the introduction and the method sections.

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
- Repertoire-based damage recovery, gradient-free but driven by a task-performance signal: a behaviour map built by MAP-Elites before deployment and searched by Bayesian optimisation after damage ([Intelligent Trial and Error, Cully et al., 2015](https://arxiv.org/abs/1407.3501)); the same without resets, with a model correcting each stored behaviour's predicted outcome ([Reset-free Trial and Error, Chatzilygeroudis, Vassiliades & Mouret, 2018](https://arxiv.org/abs/1610.04213)); and selection among several repertoires built for different anticipated situations ([adaptive prior selection, Kaushik, Desreumaux & Mouret, 2020](https://arxiv.org/abs/1907.07029)).
- Local gain-and-bias adaptation of single units toward a target activity distribution (intrinsic plasticity): Triesch, 2005.

**Not claimed as new:**
- Reward-free adaptation of frozen robot policies to body changes.
- Using the mismatch between commanded and observed motion as the adaptation signal. The contingency monitor follows Lee et al. and SCOUT, and the idea goes back at least to T-resilience ([Koos, Cully & Mouret, 2013](https://arxiv.org/abs/1302.0386)), which recovered from damage by avoiding behaviours whose observed outcome disagreed with a self-model's prediction.
- Gradient-free recovery from body damage as such. Repertoire-based methods (above) already do this.
- Intrinsic plasticity, or aligning activation statistics at test time, as such.

**What none of them does, as far as the checks found.**
1. Adapt a frozen pretrained control policy with **no deployment-time gradients and no meta-training**, using local homeostatic or ultrastable rules, with **no task-performance signal and no pre-built repertoire**. The directed rule uses no learned component beyond the policy itself. The Homeostat, the error-based family and Store-and-select use one linear forward model of the unperturbed body, fitted once before deployment and shared with every mechanism that uses it.
2. Characterise the behavioural dynamics of adaptation.
3. Separate switching, recalibrating and relearning, and test the last two by signature. As far as the checks found, none of the repertoire-based work undoes the damage to look for aftereffects or savings.

**Contributions, in priority order.**
1. **Track 2 (method): homeostatic test-time adaptation (HTA).** Gated, gradient-free plasticity in a frozen pretrained policy, with each monitor paired with the parameters it can move. Novelty is provisional until the full Track 2 literature check passes. HTA has to be shown competitive with gradient methods on at least some body changes. Otherwise it is framed honestly as a cheap first response that comes before gradient adaptation.
2. **Track 1 (evaluation).** A behavioural characterisation of switching mechanisms under world changes.
3. **Joint.** The mechanism-family × change-type interaction (HB5), and the recalibrate-versus-relearn dissociation (HB6, HB7). At the action interface, four update rules act on the same parameters from the same signal: online identification (adaptive inverse), error correction (two-state model), ultrastable random search (Homeostat), and storage with selection (Store-and-select). Differences in their signatures can then be attributed to the rule rather than to the signal or the parameterisation.

## 3. Hypotheses (registered per track before its main run)

- **HB1 (Track 2).** Under graded body changes, HTA reaches higher post-change success than Frozen and Renorm, without reward or gradients. The improvement is predicted to come from the directed rule for observation changes and from the Homeostat for action gain loss.
- **HB2 (Track 2).** Gating matters. Under no change, HTA−gate (plasticity always on) loses success relative to Frozen, while gated HTA does not.
- **HB3 (Track 1, revised).** Mechanisms trade switching latency against return after unblocking. Specific predictions:
  - FailAvoid switches fastest and returns least.
  - The habit mechanisms (HabitMem, IDSM residual) persist longest with the blocked route and return most readily once it reopens.
  - Faster decay in HabitMem shortens its switching latency and reduces its return.
  - Each is reported as a trade-off curve over a pre-registered sweep (FailAvoid's record expiry; HabitMem's decay), not as single points.
- **HB4 (Track 1).** For every switching mechanism, post-change success increases with the base policy's repertoire (from Project A). This is tested by regressing on repertoire with in-distribution competence as a covariate and base-policy seed as a random effect. This test is associational. For the memory-based mechanisms (kNN, HabitMem, IDSM residual), repertoire is also manipulated directly by removing one route's records from the seeded memory (Section 7.2).
- **HB5 (joint, revised).** In the families × {world change, graded body change} design, the improvement over Frozen depends on family and change type: the homeostatic and error-based families improve on graded body changes and not on world changes; the habit family does not improve on graded body changes. Whether the habit family improves on world changes within the budget is left to HB3. The prediction is an interaction, no longer a full crossover. It is tested as a pre-registered interaction contrast on (mechanism − Frozen), with mechanism nested in family. With the selected mechanisms per family, the claim is about these families as sampled under the membership criteria of Section 9, not about every possible member.
- **HB6 (joint, revised).** Under reversal (action inversion):
  - the error-based family (two-state model, residual fine-tune) does not improve on Frozen, and is predicted to do worse, because each correction increases the error;
  - the directed rule produces little or no improvement;
  - the Homeostat and Store-and-select improve on Frozen, more slowly than under graded changes.
- **HB7 (joint, revised).** Signatures on removal and re-exposure:
  - When a graded body change is undone, the error-based family and the Homeostat show aftereffects; the two-state model shows savings on re-exposure.
  - When the reversal is undone, the Homeostat, which overwrites, shows a large aftereffect.
  - Store-and-select shows no aftereffect beyond its selection latency, and re-adapts faster on re-exposure than on first exposure.

HB3, HB5, HB6 and HB7 are the most speculative. Clear nulls, shown by equivalence tests, count as answers.

## 4. Environments and embodiment ladder

| Rung | Environment | Use | Status |
| --- | --- | --- | --- |
| 1 | PointMaze M1 and M2 (shared with Project A); M0 for tuning | Both tracks; minimum viable experiments | Minimum viable experiment |
| 2 | A legged body (AntMaze or a MuJoCo locomotion task) | Generality | **Deferred and to be costed.** Project A does not produce legged base policies, and goal-conditioned supervised learning is weak on AntMaze. Base policies would need a separate source. **Required before rung 2:** a repertoire-switching comparator for body changes (map-guided trial and error, Section 7.2), and a way to tell a stored but previously unused gait from new movement in the behavioural labels (Section 8.3), which are fitted only on behaviour used before the change. |
| 3 | A public pretrained locomotion policy | Track 2 at realistic scale | Later. Competing methods are tested on real legged robots and large policies, so rung 2 or 3 is needed for a main-venue paper. |

The simulator version is frozen before this project starts. If Project A falls back to its NumPy simulator, this project uses the same one.

**Known limitation of rung 1.** In a 2-D point mass, the body is nearly linear and fully identifiable, so the adaptive inverse (Section 7.4) will probably solve graded action changes and reversal almost perfectly. It is reported as a ceiling, not hidden. HTA's case rests on settings where the body cannot be identified explicitly, which is why rungs 2 and 3 matter.

## 5. Base policies

Project A's goal-conditioned supervised learning policies:
- demonstrations only;
- task-directed self-practice (repertoire expected to collapse);
- frontier-novelty practice (repertoire expected to be preserved).

Each is frozen at deployment. Track 2 needs only the demonstrations-only policy, which exists early in Project A's Stage 2. HB4 needs the two self-practice policies.

The policy outputs categorical action bins, decoded to a continuous 2-D command. Everything in this document called "the policy's command" is that decoded command.

Seeds are nested: each mechanism runs several deployment seeds on each base-policy seed. The analysis treats the base-policy seed as a random effect, which avoids pseudo-replication.

## 6. Perturbations

| Class | Perturbation | Track | Role |
| --- | --- | --- | --- |
| World | Route block on the agent's most-used route (Project A's P1); unblock after a fixed delay | 1, joint | Primary. Also run for every Track 2 mechanism as a **false-alarm control**: body-change monitors should not respond to it. |
| World | Fixed block, the same for every policy (Project A's P1-fixed) | 1 | Secondary; same maze for all |
| Graded body | **Observation miscalibration**: offset and gain on position | 2, joint | **Primary.** The offset is invisible to any displacement-based error, since a constant offset doesn't change velocity; only the gain is visible. This is a predicted dissociation between the directed rule and the error-based family. |
| Graded body | **Action gain loss** (×0.5–0.7, chosen so the interface correction does not saturate the action bounds) | 2, joint | **Primary** |
| Reversal | **Action inversion**: sign flip of one or both axes | 2, joint | **Primary** for HB6 and HB7. Needs the contingency monitor (Section 7.3). |
| Body | Added action noise | 2 | Robustness probe only. Recalibration cannot remove noise, so it supports no recalibration claim. |
| Control | No change | All | Cost of adapting |

**Removal and re-exposure (for HB7).** In every body-change stream, the change is undone at a fixed point (to measure aftereffects), then re-applied later (to measure savings). Timings are fixed in the pre-registration.

## 7. Deployment protocol and mechanisms

### 7.1 Protocol (identical for every mechanism)

- **Stream.** Deployment is one continuous stream of K episodes with start–goal pairs from a fixed seeded list. The change begins at a fixed episode and step and persists. It is undone, and for body changes re-applied, at fixed later points (Section 6). Memory and plastic parameters carry across episodes.
- **Budget.** Every mechanism gets the same post-change environment steps B. Gradient-based mechanisms use the same gradient-steps-per-environment-step ratio as Project A's training, with a fixed learning rate.
- **Computational cost.** Wall-clock time per step, peak memory and the number of forward and backward passes are logged for every mechanism and reported alongside performance.
- **Tuning.** Every mechanism gets the same number of tuning trials, on M0 and on perturbation magnitudes not used in evaluation. All settings are then frozen.
- **Stall detector.** One shared, pre-registered detector, built from observations only. It fires when net displacement over the last W steps falls below δ. It uses no map, no goal distance and no reward. (v0.3 also used distance to the goal; that clause is removed because it is essentially PointMaze's dense reward and is computed from observations that may be miscalibrated.)
  - The detector is available to every mechanism.
  - Mechanisms that use it are marked in the tables.
  - kNN has a gated variant (kNN+gate) so that HabitMem's gating has a fair comparison.
- **Action interface.** Every mechanism that corrects actions does so at one shared interface between the policy and the body:

  a_env = clip(M · a_policy + c)

  where a_policy is the policy's command, M is a 2 × 2 matrix that starts at the identity, c is an offset that starts at zero, and clip applies the environment's action bounds. Each mechanism's row says which parts of M and c it changes. Mechanisms that do not use the interface leave it at the identity.
- **Command copy and intended movement.** Monitors and error-based mechanisms read a_policy, the policy's command *before* the interface. The intended movement is v_int = F(a_policy), where F is the shared forward model. The realised movement v_obs is the velocity computed from observed positions. Because the interface lies between the command copy and the body, interface parameters can change the relation between v_int and v_obs; parameters inside the policy cannot.
- **Forward model.** One shared, pre-registered linear model F of the unperturbed body, fitted on Frozen-policy rollouts before deployment: the expected velocity change given the command. It is frozen at deployment. Every mechanism that uses it is marked.
- **Pairing rule.** A monitor may drive only parameters that lie inside the loop it monitors (Ashby's condition for ultrastability). Input-side monitors, which watch unit activations, drive per-unit parameters inside the policy. The contingency monitor, which watches the relation between v_int and v_obs, drives the action interface.
- **Monitor thresholds.** Out-of-range thresholds for every monitor are set from unperturbed rollouts, using prediction-error thresholding in the style of active-inference fault detection (Baioumy et al., 2021). They are fixed before evaluation. Each monitor's false-alarm rate under no change and under the route block is reported.
- **Signals.** Each mechanism's table row lists exactly which signals it uses. No mechanism is told the perturbation type or the moment of change.

### 7.2 Habit family and Track 1 switching mechanisms (world changes)

**Seeded memory.** Every memory-based mechanism (kNN, HabitMem, IDSM residual) starts deployment with the same memory, built from Frozen-policy rollouts over all goals before deployment, so that alternative routes the policy already has are represented. For the repertoire manipulation (HB4), one route's records are removed from this seed.

| Name | Mechanism | Signals used | Gradient-free | Stage |
| --- | --- | --- | --- | --- |
| Frozen | No adaptation | — | — | Reference |
| FineTune | Online goal-conditioned fine-tuning on post-change experience, hindsight-relabelled | Own trajectories, goals | No | Reference |
| kNN / kNN+gate | Non-parametric retrieval over the agent's own (state, goal, action) records, no plasticity; the gated variant retrieves only while stalled | Own records (+ stall) | Yes | Track 1 |
| FailAvoid | Adapted from Du et al. On stall, record the actions taken at the stall state. Resample policy candidates away from recorded actions in action space (no dynamics model). Stall detection uses a progress estimator trained on the demonstrations (disclosed). Records expire after a swept horizon. Related: T-resilience (Koos, Cully & Mouret, 2013). | Stall, demonstration-trained progress estimator | Yes | Track 1 |
| **HabitMem** (habit) | Kernel-attention memory of the agent's own (state, goal, action) entries. Entries strengthen with use and decay with disuse, at all times. The stall signal gates when retrieval influences the action, not when plasticity happens. Predicted to persist: at a stall, the strongest nearby entries are the blocked route's. | Own records, stall | Yes | Screen |
| **IDSM residual** (habit) | A faithful Iterant Deformable Sensorimotor Medium (Egbert & Barandiaran, 2014) layered on the frozen policy. It records sensorimotor nodes along the trajectory, reinforces them by revisiting, lets them decay, and adds a residual motor term. Predicted to persist under world changes, and to re-enact rather than recalibrate under body changes. | Own trajectory | Yes | Screen |
| HabitMem−R, −D, −G | HabitMem without reinforcement, without decay, without gating (retrieval always on) | As above | Yes | Track 1 ablations |
| **VISSA hybrid** (habit–homeostat hybrid) | IDSM extended with a local (1+1) evolution strategy that changes nodes when the sensorimotor pattern stops being viable (Egbert, 2022). The habit family's route to switching through a viability signal. Reported separately; not counted in a family for HB5. | Own trajectory, viability score | Yes | Screen |
| **Map-guided trial and error** (reference) | Small analogue of Intelligent Trial and Error (Cully et al., 2015). Before deployment, build a map from the frozen policy: command each of a set of via-cell waypoints, then the goal, and record success and time (the waypoint is the behaviour descriptor). After a stall, choose waypoints by Gaussian-process upper-confidence-bound search, with the map as the prior mean. The canonical gradient-free switching method, and it exploits exactly the repertoire that HB4 varies. Uses a task-performance signal (disclosed). | Map, stall, success and time per trial | Yes | Stage 3 |
| FastWeight | Linear fast-weight layer updated by a self-supervised or Hebbian rule (small test-time-training analogue) | Own trajectories | No (local gradient) or yes (Hebbian) | Stage 4 |
| SuccessSteer | Bias sampling toward retrieved successful segments | Own records, success flag | Yes | Stage 4 |

**Why HabitMem's gating changed.** In v0.3 the stall signal gated plasticity. Strengthening entries with use while stalled strengthens the stalled behaviour itself, so the mechanism would perseverate by construction while being predicted to switch. Rather than redesign the mechanism to fit the prediction, v0.4 keeps use-dependent learning faithful and revises the prediction (HB3).

### 7.3 Homeostatic family (body changes)

| Name | Mechanism | Signals used | Gradient-free | Stage |
| --- | --- | --- | --- | --- |
| **HTA, directed rule** | **Input-side monitor.** Unit ranges [lo_j, hi_j] are recorded on unperturbed rollouts of the frozen policy (2.5–97.5% quantiles), not on the demonstrations, whose narrow range would trigger plasticity without any change. At deployment, each unit tracks a running average of its activation. Out-of-range units adjust their own gain g_j and bias b_j toward their range (intrinsic plasticity, after Triesch, 2005). Output-layer logit gains and biases are included, because hidden-layer changes alone may not move the categorical action bins. | Unit activations, source ranges | Yes | Screen |
| **Homeostat** | **Contingency monitor at the action interface.** Over a sliding window, it estimates the per-axis gain and sign of v_obs regressed on v_int, a normalised relation, so a robot that stops moving does not count as back in range. Its range is recorded on unperturbed rollouts. While the relation is out of range, M and c take small random steps; they are kept once it returns to range (Ashby's ultrastability; Di Paolo, 2000). Recovery from reversal requires M to pass through a sign change, so it is predicted to be slow. The command copy is *not* a policy input, which avoids the imitation-learning "copycat" problem. | Command–outcome relation, forward model | Yes | Screen |
| **HTA (combined)** | The directed rule inside the policy and the Homeostat at the action interface, each driven by its own monitor | Both | Yes | Screen |
| **Homeostatic synaptic plasticity** | One small plastic layer inserted after the input layer of the frozen policy (initialised to identity), so it lies inside the input-side loop. Its weights change by a Hebbian-type rule only while its units are out of range (Di Paolo, 2000; Iizuka & Di Paolo, 2007; Barandiaran & Di Paolo, 2010). | Unit activations, source ranges | Yes | Screen |
| HTA−gate | HTA with plasticity always on | As HTA | Yes | Ablation |
| HTA−global | One shared gain and bias instead of per-unit parameters | As HTA | Yes | Ablation |

**What makes an observation change recoverable from activations.** For an observation offset c, a first-layer unit's pre-activation becomes w·(x + c) + b, and a bias change of −w·c undoes it exactly. For an observation gain, per-unit gain and bias changes undo it approximately. What lets the directed rule find these corrections is an assumption: that the distribution of states the robot visits after the change is close to the distribution it visited before, so a shift in a unit's average activation reflects the sensor, not the robot going somewhere new. The maze, which bounds where the robot can be, supports this. The assumption is tested by comparing the offset the directed rule implicitly corrects with the true offset.

**Action changes.** Action gain loss and inversion leave the policy's inputs unchanged; any change in activations comes indirectly, through the robot visiting different states. The directed rule is therefore predicted to do little for them. They are the Homeostat's domain.

**Internal–behavioural coupling, the open research problem.** Restoring internal statistics does not by itself imply restoring behaviour. In the original homeostatic-adaptation work the link between internal stability and behaviour was built in while the controller was evolved, not assumed (Di Paolo, 2000; to be confirmed against the full text in the Track 2 literature check). The minimum viable experiment uses no coupling objective, so the directed rule is predicted to help only where the mechanism above applies. Stage 2 tests pretraining objectives that tie internal ranges to competence.

**Two mechanistic checks, before the screen.**
1. **Do the monitors move?** With the Frozen policy, measure each monitor under each perturbation, under no change and under the route block. A pairing of perturbation and mechanism is dropped if the perturbation does not move the mechanism's monitor. False-alarm rates are recorded.
2. **Can the parameters restore movement?** With the Frozen policy, set each mechanism's parameters by hand to the known correction (for example, M = the inverse of the action change; biases of −w·c for an observation offset) and measure whether success and movement recover. This shows whether each parameterisation *can* solve each change before asking whether its update rule finds the solution.

**Reported separately for every homeostatic mechanism:** whether the change was detected; whether the monitored quantity was restored; and whether task competence recovered.

### 7.4 Error-based family and references

| Name | Mechanism | Signals used | Gradient-free | Stage |
| --- | --- | --- | --- | --- |
| **Two-state adaptation** (error-based) | Per-axis interface gain and offset: M = diag(g), with g_i = 1 + x_f,i + x_s,i, and c_i = y_f,i + y_s,i. The error is e = v_int − v_obs. Each state updates as x ← A·x + B·e_i·a_policy,i (gain) and y ← A·y + B·e_i (offset), with retention A (fast: lower; slow: near 1) and learning rate B (fast: higher; slow: lower) (Smith, Ghazizadeh & Shadmehr, 2006). | Command–outcome error, forward model | Yes | Screen |
| **Residual fine-tune** (error-based) | Small analogue of Lee et al. (2026): gradient fine-tuning of the policy toward the pseudo-target a_policy + F⁻¹(v_int − v_obs) | Command–outcome residual, forward model | No | Screen |
| Two-state, single rate | One state instead of two (ablation; tests whether the fast/slow split matters for savings) | As two-state | Yes | Ablation |
| **Adaptive inverse** (reference, ceiling) | Fits the full 2 × 2 map from a_env to v_obs online by recursive least squares with forgetting, and sets M to its inverse. Identifies the body's sign rather than assuming it, so it is not counted in the error-based family. | Command–outcome pairs | Yes | Reference |
| Renorm | Standard test-time renormalisation: re-standardise each unit's pre-activation with running test statistics, ungated | Unit activations | Yes | Reference |
| EntTTA | Entropy minimisation over gain and bias (after Xu et al., 2025) | Policy outputs | No | Reference |
| **Forward-only alignment** | Analogue of FOA (Niu et al., 2024): CMA-ES on gains and biases to match source activation statistics | Unit activations, source statistics | Yes | Reference |

**Why the error-based family fails under reversal.** The gain g can become negative, so the limitation is not what the model can represent. It is the update. Under inversion the body gives v_obs = −g·a_policy, so e = (1 + g)·a_policy and the gain update has the sign of (1 + g)·a_policy², which is positive whenever g > −1: each correction moves g away from the solution g = −1. The residual fine-tune's pseudo-target assumes the same sign and pushes the same way. This matches the account of why error-based adaptation fails under mirror reversal (Yang, Cowan & Haith, 2021).

HTA uses statistics from unperturbed rollouts; Renorm does not. This difference is disclosed. HTA−gate separates the effect of gating from the effect of having source statistics. Giving Renorm the source statistics would make it the same method as HTA−gate.

### 7.5 Store-and-select (relearning test; new in v0.4)

| Name | Mechanism | Signals used | Gradient-free | Stage |
| --- | --- | --- | --- | --- |
| **Store-and-select** (hybrid) | A small library of interface settings (M_k, c_k), starting with the identity. Each entry implies a model of the body, v̂_k = F(M_k⁻¹ · (a_env − c_k)), so every entry can be scored against recent (a_env, v_obs) pairs, including entries not in use. If the best-scoring entry's error is within range, it is selected. If no entry is within range, the Homeostat's random search runs from the current entry; when the relation stays in range for T steps, the result is stored as a new entry. Capacity is small (default 4); the least-used entry is evicted. After modular and contextual-inference accounts of motor memory (Wolpert & Kawato, 1998; Haruno, Wolpert & Kawato, 2001; Heald, Lengyel & Wolpert, 2021). The robotics precedent for choosing among stored priors online is adaptive prior selection (Kaushik, Desreumaux & Mouret, 2020); its exact selection rule is to be checked against the full text in Stage 0b. | Command–outcome relation, forward model | Yes | Screen |

**Predictions.** On first exposure to reversal it behaves like the Homeostat: slow. When the reversal is undone, the identity entry scores best again and is selected, so there is no aftereffect beyond the selection latency. On re-exposure, the stored inverted entry is selected, so re-adaptation is much faster than on first exposure. Under graded changes the same pattern is predicted, so Store-and-select also tests whether storage changes the aftereffect for recalibration.

Store-and-select is a hybrid: it creates controllers by homeostatic search and switches between them by selection. It is reported separately and not counted in a family for HB5.

### 7.6 Predicted dissociations, written down now

| Change | Two-state adaptation | Residual fine-tune | HTA, directed rule | Homeostat | Store-and-select | Adaptive inverse |
| --- | --- | --- | --- | --- | --- | --- |
| Action gain loss | Corrects it; aftereffect on removal | Corrects it, more slowly; aftereffect | Little or none (inputs unchanged) | Corrects it, slowly; aftereffect | As Homeostat on first exposure; no aftereffect; fast on re-exposure | Corrects it; brief transient on removal |
| Observation offset | Blind to it (no displacement error) | Blind to it | Responds; corrects it to the extent the visitation assumption holds | Blind to it (no contingency change) | Blind to it | Blind to it |
| Observation gain | Corrects the sensed relation, not the sensor; effect on success open | As two-state | Responds (inputs rescale) | As two-state, slowly | As Homeostat | As two-state |
| Action inversion | Pushes the wrong way | Pushes the wrong way | Little or none | Slow, possible; large aftereffect on removal | Slow first time; no aftereffect; fast on re-exposure | Corrects it |
| Route block (world change) | No response expected | No response expected | Should not respond; false alarms measured | No response expected | No response expected | No response expected |

"Corrects the sensed relation, not the sensor": under an observation gain, an interface correction makes *observed* movement match the intended movement, which leaves *true* movement scaled. Whether that restores success is not predicted in advance.

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
| Return | Among runs that switched only: probability of returning to the original shorter route within W steps of unblocking | 1 |
| Decay | Usage of the abandoned route over time since abandonment | 1 |
| **Aftereffect** | After a body change is undone: the signed error between realised and intended movement over the first N steps, relative to the pre-change baseline. A significant opposite-signed error indicates overwriting. | 2, joint |
| **Savings** | Ratio of recovery latency on the second exposure to the same change to latency on the first | 2, joint |
| **Selection latency** (Store-and-select) | Steps from removal or re-exposure until the matching stored entry is selected. Aftereffects are measured after this window as well as over the first N steps. | 2, joint |
| **Monitor diagnostics** | For each monitor: detection (did it leave range after the change), restoration (did the monitored quantity return to range), recovery (did task competence return), and false alarms under no change and under the route block | 2 |
| Stability | Drift in behaviour and in success under no change | All |
| Repertoire dependence | All of the above against base-policy repertoire, with in-distribution competence as a covariate; plus the direct repertoire manipulation for memory-based mechanisms | 1 |
| Computational cost | Wall-clock time per step, peak memory, forward and backward passes | All |

The return measure was called "reversal" in v0.3; it is renamed to avoid confusion with the reversal perturbation.

### 8.3 Behavioural characterisation: how the robot adapts

Each post-change trajectory window gets **two separate labels plus a success flag**. Windows are of equal length before and after the change.

1. **Route class.** North, south or neither, from the planner's cell sets (Project A's `src/envs/mazes.py`).
2. **Motor cluster.** Computed in *outcome space*: realised displacement, speed profile, heading-change and curvature statistics, and wall-contact rate.
   - Commanded actions are excluded. After an action inversion, any adapted policy would otherwise look new by definition.
   - Clusters are fit on pre-change windows. A window is "outside" the pre-change repertoire if its distance to the nearest cluster exceeds the 95th percentile of distances for held-out pre-change windows.
   - **Imposed versus learned movement (new in v0.4).** A body change alters movement even when nothing adapts; under gain loss, the Frozen policy simply moves more slowly. Clusters are therefore also fit on Frozen-policy windows under the same perturbation. A window counts as new movement only if it lies outside both the pre-change clusters and the Frozen-under-perturbation clusters. Movement that differs from pre-change but matches Frozen under the same perturbation is labelled **imposed**.
   - The clustering method and the number of clusters are fixed in the pre-registration, and sensitivity across cluster counts is reported.
   - Descriptor choice is informed by computational sensorimotor-contingency work (Laflaquière et al.).

**Derived categories.**

| Category | Route | Motor cluster | Example |
| --- | --- | --- | --- |
| Recovered | Old | Old | Back to the pre-change behaviour |
| Rerouted | New | Old | Takes the other arm with familiar movement |
| Compensated | Old | New | Same route, new way of moving |
| Novel | New | New | New route and new movement |

Windows labelled imposed are reported as their own category. Each category is reported with and without success.

**Outputs.**
- Per-mechanism distributions over these categories.
- The number of new clusters and their distance from the pre-change repertoire.
- Hand-checked exemplar trajectories and videos.

**Status.** Descriptive and exploratory.

## 9. Analysis

- **Family membership, fixed before the screen (new in v0.4).**
  - **Habit:** changes only as a function of the agent's own use (frequency and recency of its behaviour); uses no error or outcome signal.
  - **Homeostatic:** changes only while a monitored variable is outside a range recorded in normal operation, and stops when it returns; has no target outcome.
  - **Error-based:** changes in proportion to a signed error between intended and realised outcome, with a correction direction that assumes the unperturbed body's sign.
  - **Hybrids** (VISSA, Store-and-select) combine criteria. They are screened and reported, but not counted in a family for HB5.
- **Screening, then confirmation.**
  - **Screening pilot:** every mechanism marked "Screen" plus all references; 10 seeds; route block, observation miscalibration, action gain loss, action inversion and no change.
  - **Selection rule, fixed before the screen (revised in v0.4).**
    1. **Viability gate.** A mechanism goes forward only if its audit passes, its success under no change is within an equivalence margin of Frozen (margin fixed before the screen), and it improves on Frozen on at least one change type.
    2. **Ranking.** Within each family, the two viable mechanisms with the highest mean area under the post-change success curve, averaged over change types with equal weight per type. Tiebreak: stability under no change. If a family has two or fewer viable members, all go forward.
    3. **Hybrids** go forward if viable. All references are carried forward.
  - The rule deliberately does not use each family's predicted change type, so the selection cannot favour the family × change-type hypothesis. Averaging across types favours generalists over specialists; that bias works against HB5, which is the safe direction for a confirmatory test.
  - Screening results are reported in full and labelled exploratory. Confirmatory claims cover only the selected mechanisms.
- **Pre-registration on OSF** per track, after the screen and before that track's main run.
- **Paired and nested design.** Within a deployment seed, all mechanisms share the base policy, the change onset and the noise streams.
- **Pretraining crossed with mechanism (Stage 2).** If coupling objectives are added to pretraining, every mechanism is run on both the standard and the coupled base policy, so pretraining is not confounded with mechanism.
- **Seeds.** Power analysis from the screening pilot; default 30 deployment seeds per cell.
- **Statistical methods.**
  - Latency: survival analysis (Kaplan–Meier curves, Cox models, restricted mean survival time).
  - Binary outcomes: logistic mixed models with base-policy seed as a random effect.
  - HB4: associational regression on repertoire; for memory-based mechanisms, the repertoire manipulation is analysed as a within-seed contrast.
  - HB5: interaction contrast with mechanism nested in family.
  - Aftereffects: signed-error tests against the pre-change baseline.
  - Holm correction within each track's confirmatory family.
  - Equivalence tests for null claims.

## 10. Plan and gates

| Stage | Work | Gate |
| --- | --- | --- |
| 0a | Literature check, Track 1 ✔ | ✔ |
| 0b | Literature check, Track 2. Preliminary pass done 26 October 2026. Remaining: full texts of Di Paolo (2000) and Iizuka & Di Paolo (2008), especially how the coupling between internal stability and behaviour was obtained; Lee et al. (2026), SCOUT and FOA; plasticity applied to pretrained deep policies (2023–26); modular and contextual-inference motor memory as applied to robot control; repertoire-based damage recovery (full texts of Chatzilygeroudis et al., 2018 and Kaushik et al., 2020); GRACE (arXiv 2607.21661); re-check Track 1 against September–October 2026 papers. | HTA not already published for control policies |
| 1 | **Check, build and screen** on rung 1 with the demonstrations-only base policy: shared forward model and action interface; the two mechanistic checks (Section 7.3); every "Screen" mechanism plus references; 10 seeds | Some homeostatic mechanism recovers measurably more than Frozen and Renorm on a graded body change. Otherwise Track 2 is parked and Track 1 proceeds. |
| 2 | Track 2 main: selected mechanisms, Store-and-select, coupling objectives, ablations, aftereffects and savings, behavioural characterisation, pre-registered run | HB1, HB2, HB6, HB7 decided |
| 3 | Track 1 (Frozen, FineTune, kNN, FailAvoid, map-guided trial and error, selected habit mechanisms, VISSA if selected, ablations, repertoire manipulation) on all three base policies; sweeps; battery; characterisation | HB3 and HB4 decided |
| 4 | Joint family × change-type design (HB5); FastWeight and SuccessSteer if warranted; rung 2 if costed; report | — |

**Order:** Project A's minimum viable experiment → 0b → 1, then 2 and 3 as the results warrant.

**Scope tripwire.** The two tracks stay in one study unless scope demonstrably grows beyond what the plan can carry. The test: if the Stage 1 screen is not complete by a date fixed when Project A's pilot finishes, or if the Stage 1 gate fails, the split into separate studies is reconsidered.

**Development process.** One agent implements and a separate agent audits each stage. Audit reports are committed to `notes/audits/`. The audit checks:
- equal post-change budgets;
- that "frozen" parameters really are frozen;
- the signals each mechanism uses, against Section 7;
- that each monitor drives only parameters inside its own loop (Section 7.1);
- that the command copy is taken before the action interface;
- that no mechanism sees the perturbation type or the onset;
- that nothing claimed to be reward-free uses reward;
- that the screening selection rule was applied as written.

## 11. Risks

- **HTA may restore internal statistics without restoring behaviour** (the coupling problem). Mitigations: the two mechanistic checks, the monitor diagnostics, the screening gate, and Track 1 as a standalone result.
- **A monitor may drive parameters that cannot change what it monitors.** This was the case for the Homeostat in v0.3. Mitigations: the pairing rule, the command-copy definition, the hand-set correction check, and the audit.
- **The directed rule's visitation assumption may fail**, because a failing robot visits different states. Mitigation: compare the implied correction with the true offset; report where it breaks down.
- **The adaptive inverse may dominate on rung 1.** Expected and reported. The claim about HTA rests on the family dissociations and on rungs 2–3, not on beating an explicit model in a 2-D body.
- **Relearning may not dissociate from recalibration in a 2-D body.** Both may produce similar success curves. Mitigation: the claim rests on aftereffect and savings signatures, not on success.
- **More mechanisms mean more forking paths.** Mitigations: membership criteria and selection rule fixed before the screen, screening labelled exploratory, Holm correction.
- **Small analogues may not represent large-model systems.** Mitigations: describe them honestly as analogues, and use the embodiment ladder once it is costed.
- **The field moves monthly.** Repeat the literature checks before each pre-registration.
- **Clustering can manufacture "novel" behaviour.** Mitigations: outcome-space descriptors, the imposed-movement reference, fixed thresholds, sensitivity analysis and hand checks.
- **Workload.** Store-and-select and VISSA add two mechanisms to the screen. Screening keeps the confirmatory set capped; rung 2 is deferred; the scope tripwire (Section 10) decides whether to split.
