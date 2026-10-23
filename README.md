# Switching or Recalibrating?

Two kinds of test-time adaptation in pretrained robot policies.

> **Status: design stage.**
> - Done: the literature check for Track 1, and the full design.
> - Next: Track 2's literature check, then implementation, which builds on the harness in [self-generated-experience](https://github.com/siddharthchundru/self-generated-experience).
> - There is no code and there are no results yet.

## Question

A pretrained robot policy is deployed, and something changes.

- **The world changes.** A route the robot relies on is blocked. Its sensors still report familiar-looking inputs; what fails is the behaviour. Adapting means **switching** to a different behaviour.
- **The body changes.** Actuators are inverted or weakened, or sensors are miscalibrated. The relation between what the robot commands and what it then senses leaves its trained range. Adapting means **recalibrating**.

**Hypothesis.** These are different problems, solved by different mechanisms.

- Mechanisms that keep a use-dependent memory of the robot's own behaviour should switch well but not recalibrate.
- Mechanisms that restore internal and sensorimotor operating ranges should recalibrate well but not switch.

The two mechanism families come from work on habits as use-dependent sensorimotor patterns ([Egbert & Barandiaran, 2014](https://doi.org/10.3389/fnhum.2014.00590)) and on homeostatic plasticity ([Di Paolo, 2000](https://sro.sussex.ac.uk/id/eprint/29695/), after Ashby). This project tests them against small analogues of current test-time adaptation methods for robot policies.

## Why this matters

Test-time adaptation of robot policies is moving fast. Examples include:

- gradient-based, critic-driven adaptation ([Alfarra et al., 2026](https://arxiv.org/abs/2610.20318));
- fast-weight memories updated during deployment ([Liu et al., 2026](https://arxiv.org/abs/2609.36720); [Jiang et al., 2026](https://research.nvidia.com/labs/gear/robottt/));
- gradient-free retrieval and steering ([Zhao et al., 2026](https://arxiv.org/abs/2605.10094); [Chen et al., 2026](https://arxiv.org/abs/2610.12185));
- resampling away from past failures ([Du et al., 2024](https://arxiv.org/abs/2406.15917)).

These papers mainly report success after a change, and some report latency. As far as a first-pass literature check found, none characterises **how** a robot adapts: how long it persists with a strategy that has stopped working, whether it returns when the original solution works again, what adapting costs when nothing has changed, and which new behaviours appear. Nor do they separate switching from recalibrating. The literature check is in [`notes/stage0-literature-gate.md`](notes/stage0-literature-gate.md).

## Design

The setting is two-dimensional maze navigation ([PointMaze](https://robotics.farama.org/envs/maze/point_maze/)). The base policies come from [self-generated-experience](https://github.com/siddharthchundru/self-generated-experience), trained under practice conditions predicted to differ in repertoire (measured, not assumed). A legged body is a possible later extension, not yet costed.

**Track 1: switching under world changes.** A behavioural comparison, on the same frozen base policies, of:

- online fine-tuning;
- non-parametric retrieval;
- failure-avoidance resampling;
- a **habit memory**: an attention memory over the robot's own experience whose entries strengthen with use and fade with disuse.

**Track 2: recalibrating under body changes.** A proposed method, **homeostatic test-time adaptation**, which adapts with no reward, no task loss and no backpropagation:

- Each unit of the frozen policy network monitors its own activity against the range it showed in normal operation.
- A separate monitor tracks the relation between commanded action and observed outcome.
- Plasticity switches on only where activity leaves its range, and only until it returns; when the command–outcome relation leaves its range, gains and biases take small random steps until it returns.

It is compared with standard test-time renormalisation and entropy-based adaptation. Whether this is new for control policies is being checked in Track 2's literature review.

**Joint test.** A 2 × 2 design of {habit memory, homeostatic adaptation} × {world change, body change}.

## Measures

- **Success**, comparable with prior work: post-change success, the area under the recovery curve, and success when nothing changes.
- **Adaptation dynamics**:
  - perseveration (persisting with a strategy that has stopped working);
  - time to switch or recover;
  - reversal after the original route reopens;
  - decay of an abandoned strategy;
  - stability under no change;
  - dependence on the base policy's repertoire.
- **How the robot adapts.** Each post-change trajectory is labelled on two separate axes:
  - **route**: old or new;
  - **movement pattern**: familiar or new, judged by what the robot actually does rather than what it commands.

  This separates *recovered*, *rerouted*, *compensated* and *novel* behaviour.

**Analysis.** Each track will be pre-registered on OSF before its main run. Latencies are analysed with survival analysis and outcomes with mixed models, with equivalence tests for null results. The full design is in [`docs/spec.md`](docs/spec.md).

## Roadmap

- [x] **Stage 0a:** literature check for Track 1
- [ ] **Stage 0b:** literature check for Track 2
- [ ] **Stage 1:** Track 2 pilot
- [ ] **Stage 2:** Track 2 main study
- [ ] **Stage 3:** Track 1 main study
- [ ] **Stage 4:** joint 2 × 2 test; a second embodiment if costed

Each stage ends in a gate that must pass before the next starts. Before a stage closes, its code is reviewed by a separate AI coding agent (not the one that wrote it); those review reports will be committed to `notes/audits/`.

## Repository

```
notes/          literature check
docs/spec.md    full design specification
```

Code arrives with Stage 1.

## Author

Siddharth Chundru · [siddharthchundru.github.io](https://siddharthchundru.github.io) · sbchundru@gmail.com
