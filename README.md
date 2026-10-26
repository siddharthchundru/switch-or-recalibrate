# Switching, Recalibrating or Relearning?

Kinds of test-time adaptation in pretrained robot policies.

> **Status: design stage.**
> - Done: the literature check for Track 1; a preliminary literature check for Track 2; the full design (spec v0.4, revised after an external review of v0.3).
> - Next: implementation, which builds on the harness in [self-generated-experience](https://github.com/siddharthchundru/self-generated-experience), then the full Track 2 literature check before any pre-registration.
> - There is no code and there are no results yet.

## Question

A pretrained robot policy is deployed, and something changes. There are three kinds of change, and each may ask for a different kind of adaptation.

- **The world changes.** A route the robot relies on is blocked. Its sensors still report familiar-looking inputs; what fails is the behaviour. Adapting means **switching** to a different behaviour the policy already has.
- **The body changes by degree.** Actuators weaken, or a position sensor drifts (offset or gain). The relation between what the robot commands and what it then senses shifts but keeps its direction. Adapting means **recalibrating**: the same behaviour, adjusted.
- **The body is reversed.** An actuator's direction is inverted. The relation between command and outcome changes sign. Adapting may mean **relearning**: acquiring a new controller.

Reversal is its own class because human motor learning treats it that way. Gains, offsets and rotations are handled by fast, error-driven adaptation that leaves *aftereffects* when the change is undone. Mirror reversal is learned from scratch instead: slower, with no aftereffects, and error-driven correction can even push the wrong way ([Yang, Cowan & Haith, 2021](https://elifesciences.org/articles/62578)).

**Recalibrating and relearning are told apart by what the robot does, not by which change it faced.** A mechanism that adjusts its existing controller in place should show an aftereffect when the change is undone. A mechanism that stores a new controller alongside the old one, and selects between them, should show none, and should adapt faster the second time. Both signatures are measured.

**Hypothesis.** Different kinds of change are handled by different families of mechanism, and the families leave different behavioural signatures.

- **Habit mechanisms**, which keep a use-dependent memory of the robot's own behaviour, should persist with a strategy after it stops working, switch slowly, and return readily when the original route reopens. They should not recalibrate or relearn.
- **Homeostatic mechanisms**, which restore internal and sensorimotor operating ranges, should recalibrate without any model of the error. The ultrastable variant may recover from reversal, slowly, by overwriting what it had.
- **Error-based mechanisms**, which correct from the gap between intended and observed outcome, should recalibrate graded changes well, but push the wrong way under reversal, because their corrections assume the body's original direction.
- **A store-and-select mechanism**, which keeps a small library of controllers and picks the one that best explains recent movement, should recover from reversal without an aftereffect.

The habit and homeostatic families come from work on habits as use-dependent sensorimotor patterns ([Egbert & Barandiaran, 2014](https://doi.org/10.3389/fnhum.2014.00590)) and on homeostatic plasticity ([Di Paolo, 2000](https://sro.sussex.ac.uk/id/eprint/29695/), after Ashby). The error-based family is the standard two-timescale model of human motor adaptation (Smith, Ghazizadeh & Shadmehr, 2006). Store-and-select follows modular and contextual-inference accounts of motor memory (Wolpert & Kawato, 1998; Heald, Lengyel & Wolpert, 2021). Membership of each family is defined by fixed criteria, so claims can be made about families rather than single mechanisms. All are compared against small analogues of current test-time adaptation methods for robot policies.

## Why this matters

Test-time adaptation of robot policies is moving fast, with several of the closest papers appearing in September and October 2026. Examples include:

- gradient-based, critic-driven adaptation ([Alfarra et al., 2026](https://arxiv.org/abs/2610.20318));
- reward-free fine-tuning from the gap between commanded and measured motion ([Lee et al., 2026](https://arxiv.org/abs/2609.37334));
- a meta-trained belief updated from action–outcome mismatch ([Li et al., 2026](https://arxiv.org/abs/2609.36107));
- a learned adapter driven by motor telemetry ([Sagar & Senanayake, 2026](https://arxiv.org/abs/2609.36182));
- fast-weight memories updated during deployment ([Liu et al., 2026](https://arxiv.org/abs/2609.36720); [Jiang et al., 2026](https://research.nvidia.com/labs/gear/robottt/));
- gradient-free retrieval and steering ([Zhao et al., 2026](https://arxiv.org/abs/2605.10094); [Chen et al., 2026](https://arxiv.org/abs/2610.12185));
- resampling away from past failures ([Du et al., 2024](https://arxiv.org/abs/2406.15917)).

So reward-free adaptation to body changes, using the command–outcome mismatch as the signal, and local gain-and-bias plasticity on its own are **not** new, and this project does not claim them. As far as the checks so far found, what is open is:

1. adapting a frozen pretrained control policy with **no deployment-time gradients and no meta-training**, using local homeostatic rules. One rule needs nothing beyond the policy itself; the others use a single linear model of the unperturbed body, fitted once before deployment and shared with the baselines;
2. characterising **how** a robot adapts: how long it persists with a strategy that has stopped working, whether it returns when the original solution works again, what adapting costs when nothing has changed, and which new behaviours appear;
3. separating switching, recalibrating and relearning, and testing the last two by their signatures.

The Track 1 literature check is in [`notes/stage0-literature-gate.md`](notes/stage0-literature-gate.md). The Track 2 check has had a preliminary pass; its memo will be added when the full check is done.

## Design

The setting is two-dimensional maze navigation ([PointMaze](https://robotics.farama.org/envs/maze/point_maze/)). The base policies come from [self-generated-experience](https://github.com/siddharthchundru/self-generated-experience), trained under practice conditions predicted to differ in repertoire (measured, not assumed). A legged body is a planned later extension, not yet costed.

**Track 1: switching under world changes.** A behavioural comparison, on the same frozen base policies, of online fine-tuning, non-parametric retrieval, failure-avoidance resampling, and habit mechanisms: a **habit memory** (an attention memory over the robot's own experience whose entries strengthen with use and fade with disuse), a residual version of the Iterant Deformable Sensorimotor Medium (Egbert & Barandiaran, 2014), and a hybrid that also changes a habit when it stops being viable (Egbert, 2022). For memory-based mechanisms, the repertoire available to them is also manipulated directly.

**Track 2: recalibrating and relearning under body changes.** A proposed method, **homeostatic test-time adaptation**, with no reward and no gradients. Its central design rule, after Ashby: each monitor changes only parameters that can affect what it monitors.

- **Inside the policy:** each unit monitors its own activity against the range it showed in normal operation. Out-of-range units adjust their own gain and bias back toward that range. This can undo a sensor offset exactly, but cannot see an actuator change, which leaves the inputs unchanged.
- **Between the policy and the body:** a second monitor tracks the relation between the movement the policy intended and the movement that happened (following Lee et al. and Li et al., above). When it leaves its normal range, a small correction between policy and body takes random steps until the relation returns (Ashby's ultrastability).
- Plasticity is switched on only while something is out of range.

At that interface, four update rules act on the same parameters from the same signal: online identification of the body, two-timescale error correction, ultrastable random search, and **store-and-select**, which keeps the controllers it finds and reselects them. Further references include standard test-time renormalisation, entropy-based adaptation, a gradient fine-tune on the command–outcome signal, and a gradient-free activation-alignment method. In a 2-D point mass the online identification will probably solve every actuator change almost perfectly; it is reported as a ceiling, and it is why later stages move to bodies that cannot be identified so easily.

Before any screening, two checks: that each perturbation actually moves the monitors meant to detect it, and that each mechanism's parameters can restore movement when set by hand to the right correction.

**Screening, then confirmation.** Every candidate mechanism runs in a short screening pilot. A rule fixed in advance picks which go into the pre-registered confirmatory runs: a viability gate, then the best two per family by average performance across all kinds of change. The rule deliberately ignores which kind of change each family is predicted to handle, so it cannot tilt the result toward the hypothesis. Screening results are reported in full and labelled exploratory.

**Joint test.** A design of mechanism family × kind of change, testing whether the benefit of each family depends on the kind of change.

## Measures

- **Success**, comparable with prior work: post-change success, the area under the recovery curve, and success when nothing changes.
- **Adaptation dynamics**:
  - perseveration (persisting with a strategy that has stopped working);
  - time to switch or recover;
  - return to the original route after it reopens;
  - decay of an abandoned strategy;
  - **aftereffects**: errors in the opposite direction just after a body change is undone, the signature of adjusting in place;
  - **savings**: faster adaptation the second time the same change appears;
  - for each monitor, whether it detected the change, whether what it monitors was restored, whether the task recovered, and how often it fired when nothing had changed;
  - stability under no change;
  - dependence on the base policy's repertoire;
  - computational cost.
- **How the robot adapts.** Each post-change trajectory is labelled on two separate axes:
  - **route**: old or new;
  - **movement pattern**: familiar or new, judged by what the robot actually does rather than what it commands, and compared with how an unadapted policy moves under the same change, so that changes the body imposes are not counted as learned.

  This separates *recovered*, *rerouted*, *compensated* and *novel* behaviour.

**Analysis.** Each track will be pre-registered on OSF after screening and before its main run. Latencies are analysed with survival analysis and outcomes with mixed models, with equivalence tests for null results. The full design is in [`docs/spec.md`](docs/spec.md).

## Roadmap

- [x] **Stage 0a:** literature check for Track 1
- [ ] **Stage 0b:** literature check for Track 2 (preliminary pass done; full texts pending)
- [ ] **Stage 1:** mechanistic checks, then build all candidate mechanisms and run the screening pilot
- [ ] **Stage 2:** Track 2 main study (recalibrating and relearning)
- [ ] **Stage 3:** Track 1 main study (switching)
- [ ] **Stage 4:** joint family × kind-of-change test; a second embodiment if costed

Each stage ends in a gate that must pass before the next starts. Before a stage closes, its code is reviewed by a separate AI coding agent (not the one that wrote it); those review reports will be committed to `notes/audits/`.

## Repository

```
notes/          literature check
docs/spec.md    full design specification (v0.4)
```

Code arrives with Stage 1.

## Author

Siddharth Chundru · [siddharthchundru.github.io](https://siddharthchundru.github.io) · sbchundru@gmail.com
