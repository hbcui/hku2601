# Decision-centric imagination for the route-only AMHS world model

## Why the objective changes

The existing map-pair pilot trained at 2, 5, and 10 **frames** and reported
roughly 9.5 m position error after a 60 s open-loop rollout. It also replayed
future routing actions selected on the *true* simulator state. That is a useful
conditional forecasting diagnostic, but it is not an autonomous control
experiment. A 60 s exact state trajectory is not the objective of the new
controller. The objective is to rank route choices reliably over a calibrated
short imagination horizon, execute one choice, observe again, and replan.

## What is borrowed from the papers

| Reference | Mechanism in the paper | Here |
|---|---|---|
| [PlaNet](https://arxiv.org/pdf/1811.04551) | Recurrent stochastic state-space model; finite-horizon planning in learned latent state; replan after observing the next state | Keep graph RSSM and reanchor its belief at every real observation. Advance the learned ledger only for a short candidate horizon. |
| [DreamerV3](https://arxiv.org/pdf/2301.04104) and [code](https://github.com/danijar/dreamerv3) | Short latent imaginations plus a learned value bootstrap; continuation matters | Use a held-out-trained terminal value after short imagination. A later model revision should train reward/continuation heads jointly with the dynamics. Do not treat a 15-step actor imagination as a guarantee of exact 60 s observations. |
| [TD-MPC2](https://arxiv.org/pdf/2310.16828) and [code](https://github.com/nicklashansen/tdmpc2) | Optimize a short latent trajectory with terminal value, execute the first action, replan on the next observation | Evaluate legal route candidates, choose one current route write, and replan on the next observed full system state. Candidate search is a controller operation, not the world-model transition. |
| [Options framework](https://people.cs.umass.edu/~barto/courses/cs687/Sutton-Precup-Singh-AIJ99.pdf) | A temporally extended action has initiation, internal continuation, and termination | A route commit initiates a durable route option. Its node sequence, cursor, target and task assignment remain explicit ledger state until completion or replacement. |

The match is at the **mechanism** level. The present model is not a faithful
DreamerV3 or TD-MPC2 reproduction: its categorical latent is currently used
deterministically in free rollout; it has no jointly trained reward or
continuation head; and the first terminal value is trained separately on
factual trajectories. Those differences are experimental risks, not claimed
paper results.

## Causal state and time

At an observed decision boundary, construct the full graph of OHT positions
on directed RailPaths, persistent routes and cursors, active task commitments,
resource holders/queues, and topology features. A route decision changes the
ledger for its named OHT, then the graph attention model predicts the effects
on **all** OHTs, tasks, and resources. Task arrivals are exogenous. Dispatch,
resource grants, service completion and motion are modeled outcomes. The
world model never accepts those outcomes as control actions.

Use the actual 30 Hz transition as the current model's integration unit. The
planner's `horizon_frames` is therefore a number of model transitions, not a
claim that a 30 s gap can be advanced in one step. The next controller call
must use a new observation. Plan while a route option remains active: the
route decision has long-lived effects even though the imagination is short.

## Code contract implemented now

- `src/route_decision_planner_v1/decision_imagination.py`: legal directed-route candidates, short
  candidate imagination, persistent route writes, same-forecast comparison,
  terminal completion value, and per-observation replanning API.
- `src/route_decision_planner_v1/train_completion_value.py`: train the behavior-policy terminal
  value on future `complete_task` events from train seeds; validate on a
  separate route seed. The event labels determine the target only and are
  never inputs to candidate imagination.
- `src/route_decision_planner_v1/test_decision_imagination.py`: checks legal alternatives, lasting
  route mutation, no observed-state mutation, fresh reanchoring, and refusal
  to pass a future route action through the task-arrival forecast interface.
- `src/route_decision_planner_v1/inspect_route_imagination.py`: read-only one-decision inspection.
  It requires a held-out value fit better than the training-mean baseline.
  If the value gain over preserving an existing route is smaller than the
  held-out value MAE, it preserves that route. This is an abstention heuristic,
  not a statistical confidence bound on a counterfactual value difference.

The planner returns the route write to execute. It does not silently execute
that write in the native simulator. An online adapter must apply the chosen
route and pass the next actual simulator observation into a new `plan` call.

## What the small pilot can and cannot establish

The first value target is **completed tasks in the next 30 s under the logged
behavior policy**. It is a bootstrap signal, not an action-value oracle.
Training it on a few seeds does not identify the causal effect of routes that
the logged controller rarely or never selected. A mean-prediction baseline and
held-out seed MAE must accompany every value fit. A random or uncalibrated
value model must not be used to claim policy improvement.

The initial FabGrid pilot (`outputs/decision_imagination_fab_grid_pilot`) used
route seeds 0/1/2 for 8 small value epochs and route seed 3 for validation.
The best held-out MAE was **2.72 tasks** over 30 s versus **14.16 tasks** for a
constant training-mean prediction, on only 16 sampled seed-3 states. The
simulator seed and task-generation configuration are shared across route
seeds, so this split tests routing variation much more than task-arrival
generalization. At frame 900, a three-frame alternative-route score differed
from preserving the route by only about **0.00037 tasks**, far below value
validation error; the controller correctly kept the existing route. This
provides no evidence that the value model detects the causal effect of a
different route.

The current tensorizer exposes only the first four ordered route nodes plus
coarse full-route scalars. A candidate that diverges only beyond those nodes
may be indistinguishable at the present observation. The candidate generator
therefore defers such a change until the persistent route cursor brings the
branch into view. A future model should encode the complete route sequence
with a topology-aware option encoder, then verify that route alternatives
create distinguishable latent transitions on paired simulator branches.

The next controlled experiment should use simulator snapshots at matched
decision states. For each state, apply alternative legal route commits while
holding simulator seed, task-arrival stream, topology and all other settings
fixed. Those branches supply causal action coverage and permit a direct
**ranking accuracy** test. Then run a closed-loop policy test: predict a
short interval, execute one selected route in the real simulator, observe,
and replan. Report completed tasks, task delay, congestion/deadlocks, route
violations, decision latency, and paired confidence intervals across routing
seeds. Keep the old factual action-replay 60 s test only as a labeled
conditional-prediction diagnostic.

## Training progression and gate

1. Fit and calibrate one-step OHT motion, route-boundary, task and resource
   transitions on held-out native data. Report event recalls by type and
   identity-bearing state errors; a low aggregate loss is insufficient.
2. Increase closed-loop *training* windows according to measured error: first
   a few frames, then 0.5 s, 1 s, and 2 s if validation remains stable. Train
   every intermediate step; oversample route commits and task lifecycle
   boundaries without reordering them. Use observation-conditioned posterior
   learning only after implementing reconstruction and KL together; adding
   KL alone does not reproduce an RSSM objective.
3. Fit reward/continuation/value jointly or with a verified compatible latent
   state, then choose the planner horizon by held-out **action-ranking** and
   latency, rather than by a preset 60 s target. Estimate uncertainty across
   an ensemble or stochastic latent samples before trusting close scores.
4. Only promote a model when closed-loop simulator outcomes beat the native D
   baseline on held-out routing seeds, without increasing invalid routes or
   deadlocks. No such promotion result exists yet.

## Existing-data imagination audit (2026-11-08)

The first audit compared three protocols on
the same three seed-3 anchors (frames 900, 1800, 2700) and their exact logged
route and task-arrival input timelines. Every number below is measured **before**
any observation reset at that frame. The model predicts internal outcomes;
future observed states are used only for scoring, except at explicit resets.
The complete historical-replay report is
`outputs/decision_imagination_fab_grid_pilot/protocol_3anchors_5s.json`.

| Elapsed | Open-loop pose MAE | Reset predicted latent each frame | True-observation reset every 1 s | Open-loop path accuracy | True-observation path accuracy |
|---:|---:|---:|---:|---:|---:|
| 0.1 s | 0.013 m | 0.012 m | 0.013 m | 99.3% | 99.3% |
| 0.5 s | 0.078 m | 0.064 m | 0.078 m | 95.6% | 95.6% |
| 1 s | 0.171 m | 0.146 m | 0.171 m | 89.8% | 89.8% |
| 2 s | 0.383 m | 0.352 m | 0.184 m | 82.7% | 90.7% |
| 5 s | 1.057 m | 1.017 m | 0.194 m | 73.1% | 90.1% |

This establishes a practical difference between open-loop conditional
forecasting and receding-horizon state estimation on these anchors. The
true-observation-reset column is **not** a five-second autonomous forecast:
it uses real observations at 1, 2, 3, and 4 seconds before predicting the
last second. Resetting only the predicted latent yields little 5-second
improvement, so latent carry alone does not explain the accumulated error.
At 5 seconds, open-loop resource-holder recall is 82.8%, versus 95.7% with
observed resets. The open-loop state rejects 8.3 factual external inputs on
average per anchor because its predicted ledger has diverged; this makes the
factual-action-replay evaluation increasingly conditional on incompatible
future inputs. The run also records 361 route-end motion caps per anchor over
five seconds. These are diagnostic occurrences, not 361 distinct OHTs.

**Causal-input caveat:** the historical transition loader classified *every*
logged `route_commit` as a route input. In seed 3 there are 1,203 route writes
marked `control_eligible=True` and 2,270 native internal route writes marked
`control_eligible=False` (mostly `end_cruising` and `driveaway_start`). The
five-second evaluation above supplied both. Across its three anchors it
supplied 25 controllable and 45 internal route writes per anchor. The latter
are not decisions the new controller can know in advance. The
`control_only` audit withheld them before the loader was fixed. At 5 s the
pose MAE changed from 1.057 to 1.075 m and path accuracy from 73.1% to
71.1%; at 1 s, pose MAE remained 0.171 m and path accuracy changed from
89.8% to 89.1%. The current loader now withholds these internal writes in
all causal protocols, and the curriculum supervises their event heads from
the endogenous labels. `control_only` remains a counting diagnostic. Even
the 0.1–0.5-second replay numbers cannot be called a verified autonomous
imagination result, because the logged eligible controller actions were
chosen against the true simulator state.

After the correction, a six-update, ten-frame causal pilot from the same
checkpoint used train seeds 0/1/2 and the same three held-out seed-3 anchors.
Against the unmodified checkpoint evaluated with internal route writes
withheld, its results were:

| Elapsed | Pose MAE before → after | Path accuracy before → after | Moving OHT recall before → after |
|---:|---:|---:|---:|
| 0.1 s | 0.013 → 0.014 m | 99.1 → 98.8% | 88.1 → 90.7% |
| 0.5 s | 0.078 → 0.085 m | 95.0 → 94.4% | 86.1 → 87.5% |
| 1 s | 0.171 → 0.183 m | 89.1 → 88.3% | 86.2 → 87.8% |
| 2 s | 0.385 → 0.402 m | 81.3 → 81.8% | 87.1 → 88.8% |
| 5 s | 1.075 → 1.086 m | 71.1 → 70.6% | 80.7 → 81.9% |

This smoke verifies that the corrected route head trains and that moving
recall rises slightly; it does **not** show improved long rollout. Six updates
on three related routing seeds are too few to judge eventual model quality.
The report is `outputs/imagination_causal_route_pilot/eval_seed3_3anchors_5s.json`.

This does not validate ranking alternative routes. Route
commits have persistent effects beyond this window, and the terminal value
trained from factual data has no counterfactual guarantee. An online planner
should execute a decision, observe the resulting system, and replan; its
task-throughput and safety effects still need paired simulator branches.

The small teacher-forcing ablation used six identical sampled ten-frame
updates from the same starting checkpoint on train route seeds 0/1/2, then
two seed-3 evaluation anchors. Re-encoding the latent after replacing the
ledger with a true observation gave 2 s pose MAE **0.315 m**, versus **0.316 m**
when the prior latent was carried; both had 85.7% path accuracy. This is
insufficient evidence that latent reset alone improves free rollout. The
ablation preceded the correction to internal-route event classification and
is retained only as a narrow diagnostic; it is not a model-selection result.

## Markov-closure and event-hazard audit (2026-11-08)

The five-second bursts initially looked like an unexplained global cruise
clock. Checking the native event order changed that diagnosis: the simulator
uses **batch task dispatch every 5 s**. A batch assignment calls
`end_cruising`, writes task lifecycle and dispatch records, and may then ask
the route controller to commit a new plan at the same simulator time. Thus
the known dispatch schedule is part of the environment state, while the
selected OHT/task pair is an endogenous outcome and the route choice is the
external control. At frame 152 of held-out route seed 3, the raw log has 12
internal route writes, 10 task starts, and 8 eligible route controls. A
frame-level world model cannot treat all these records as independent
post-control decisions.

On the complete 300-second seed-3 episode, there are 2,270 raw internal
route writes, but only **1,583** remain after discarding same-timestamp writes
overwritten by a later route write for the same OHT; 687 are transient.
The current label builder, calibration, and evaluator now distinguish these
effective writes from the raw event audit. This is a 30 Hz state-transition
choice, not a claim that the intermediate simulator hooks never occurred.

The route-event pilot also exposed a rate/calibration failure. On 180 sampled
training frames, 34 OHT-frame labels were positive, while the uncalibrated
categorical head assigned an expected **5,541** events. Its argmax-plus-ETA
gate then emitted **zero**. A fitted logit intercept (-5.285) matched the
training event count and estimated 34.2 versus 39 actual labels on the
held-out seed, but the highest-risk OHT hit none of 31 event frames. In a
two-anchor one-second free rollout, calibrated sampling produced 3.5 route
writes per anchor versus 17.5 raw observed writes, with no material gain in
pose or path accuracy. The calibration was fitted before transient-label
filtering; it is an exploratory failure diagnosis, not a deployable model.

The transition graph now exposes a configured batch-dispatch clock through
three reserved features (absolute time and sine/cosine phase), while
preserving the checkpoint feature dimension. The ledger advances dwell time,
moving age, rail-entry time, and learned acceleration instead of repeatedly
feeding stale native values. Internal-route windows and effective-route
targets are now sampled explicitly in the small curriculum. The learned
route outcome is sampled from a calibrated per-OHT hazard in a separate
evaluation protocol; its ETA head is no longer a hard veto in that protocol.
These are changes to the **state and learning contract**; no native routing
heuristic has been inserted into the world-model transition.

The graph is still not formally Markov-complete. It compresses each full
route to four ordered lookahead nodes and summary scalars, omits some native
resource queue order and OHT control state, and does not expose the exact
future-event-list phase. The stochastic latent currently uses the expected
categorical embedding in rollout, with no jointly trained posterior/KL and
no calibrated reward/continuation head. Therefore neither this code nor the
papers guarantees a faithful 60-second open trajectory. The defensible
planning contract is short conditional imagination, one route decision,
observation update, and replanning; empirical action ranking and closed-loop
simulator outcomes remain the promotion tests.

### Held-out, matched-anchor experiment

The final small pilot trained on route seeds 0/1/2 and was evaluated on held-out
seed 3, at frames 900 and 1800. Both checkpoint evaluations below use the
same current causal evaluator, true future *eligible* route controls and task
arrivals, and continuous predicted-ledger feedback. The old checkpoint has
its new clock slots zeroed because they were absent at its training time.
Each number is the mean of only two overlapping anchors; this is a diagnostic,
not a robust model comparison.

| Free horizon | Old pilot pose MAE | New pilot pose MAE | Old path accuracy | New path accuracy | Old/new route-end caps |
|---:|---:|---:|---:|---:|---:|
| 1 s | 0.146 m | 0.118 m | 90.0% | 92.0% | 95.5 / 108.0 |
| 2 s | 0.331 m | 0.287 m | 84.8% | 85.0% | 196.5 / 321.5 |
| 5 s | 0.953 m | 0.877 m | 75.5% | 73.3% | 947.5 / 1878.0 |

At 5 s the new pilot still predicts **zero** task starts versus 10.5 actual
and **zero** effective internal route writes versus 29.5 actual per anchor.
Its 5 s predicted mean OHT speed is 0.579 m/s versus 0.465 m/s observed;
the old model predicts 0.468 m/s. Improving aggregate pose MAE while losing
path accuracy and doubling route-end caps does not establish better dynamics.
The matched reports are
`outputs/imagination_causal_route_pilot/eval_current_code_seed3_2anchors_5s.json`
and
`outputs/imagination_route_hazard_clock_accel_36u_5f/eval_seed3_2anchors_5s.json`.

The marginal route-event calibration on 240 sampled observed frames per split
fits an intercept of -3.598. It predicts 37.0 effective writes on the
held-out sample versus 43 actual, but identifies the correct OHT as its top
choice on only **1 of 33 event frames**. Disabling the approximate dispatch
clock leaves top-1/top-5/top-20 event localization unchanged. Thus rate
matching does not solve the causal *which OHT* question. In a sampled 2 s
rollout the calibrated protocol writes 5.0 routes versus 9.5 effective
observed writes per anchor, and its 2 s pose MAE rises from 0.287 to
0.292 m. Calibration file:
`outputs/imagination_route_hazard_clock_accel_36u_5f/calibration_clock.json`.

A second rapid pilot corrected a training-loss normalization error: the
per-frame route-event CE had been averaged with a variable number of route
plan-token losses, so an event-rich frame diluted the event-hazard gradient.
It separates occurrence and content means, then ran 60 updates of eight
frames each with 50% teacher forcing and event-focused sampling (train route
seeds 0/1/2). Held-out seed-3 top-1 event localization rose only from 1/33
to **2/33** event frames, top-20 from 4/33 to **7/33**. Its 1 s open pose MAE
is 0.101 m and path accuracy 94.0%, but at 5 s these are 0.889 m and
73.2%; 1,583 route-end caps occur per anchor. It still predicts zero task
starts and zero internal route writes with argmax decoding. These tiny pilots
show an event-supervision bug was fixed, not that persistent dynamics is
solved. Files: `outputs/imagination_route_hazard_balanced_60u_8f/summary.json`,
`calibration_seed3.json`, and `eval_seed3_2anchors_5s.json` in that directory.
With calibrated sampling, 18.0 internal routes are written in five seconds
versus 29.5 effective native writes per anchor, but 5 s pose MAE **worsens**
from 0.889 to 0.928 m, path accuracy from 73.2% to 72.5%, and route-end
caps from 1,583 to 1,705.5. Matching a global event count while choosing
mostly wrong OHTs therefore degrades closed-loop predictions. The sampled
protocol is stochastic with one fixed RNG seed per anchor; two anchors do
not estimate a sampling confidence interval.

The next criterion is whether event-focused training improves held-out
per-OHT event ranking and reduces free-rollout route caps **together**. A
short pose-only gain does not pass it. Even if it passes, a Markov closure
audit still requires exact scheduled dispatch phase, ordered resource queues,
full persistent route/control state or a posterior belief that can infer
their missing information. The native simulator's event queue and random
generator can make two identical exported snapshots have different futures;
the exported snapshot alone therefore cannot guarantee a deterministic
Markov transition.

### November 8 event-boundary correction and diagnostic

The route-only control contract treats task arrival and the dispatcher result
(`preassign`, `clear_preassignment`, `start_task`) as external task-stream
inputs. They are applied at their logged time and remain in the persistent
ledger. Task completion, resource outcomes and native internal route writes
remain model predictions. This is a conditional rollout: using the recorded
future task stream measures system prediction given that stream. Autonomous
generation would require a separate task/dispatcher model. Earlier reports
that counted `start_task` as an endogenous prediction used a different
boundary and are not directly comparable.

A previous-frame-only event-identity probe trained on seeds 0–2 put a correct
OHT in its top 20 on 86.5% of held-out seed-3 frames containing an effective
internal route write (297 event frames). The existing RSSM head reached 18.5%
on the same frame set. On 240 uniformly sampled held-out frames (33 event
frames), the probe's event-timing AUC was 0.726. Good conditional identity
ranking therefore does not yet establish a reliable joint time-and-entity
model. Adding more persistent queue/ledger fields did not improve the ranking
on this split. The evidence is in `markov_state_audit.json` and
`event_identity_probe_calibrated_seed3.json` under
`outputs/imagination_route_hazard_balanced_60u_8f/`.

### Target imagination contract

An option-level decision is a `route_commit` for one OHT. Its route-plan
pointer and target stay in the **explicit** state until another commit, task
transition, or completion changes them; the control is not replayed as a
one-frame pulse. A task arrival is an external event whose record and
assignment relation persist. The model's stochastic state is a belief over
unobserved native timers, queue ordering and dispatch outcomes, conditioned
on the observed system ledger. At each prediction step it should produce a
joint event time/type/entity distribution and a rail-constrained motion
increment, then apply the sampled event's durable ledger delta. A route
action can affect all OHTs through the shared resource and contention graph.

The current code implements the persistent ledger and rail-constrained
increment, but still invokes the event heads at every 30 Hz microstep. It
does **not** yet implement a trained event-time distribution or a jointly
trained observation posterior. These are required before calling it an
event-driven RSSM. The desired training loop is: infer posterior from the
next actual graph; train prior/posterior agreement with categorical KL and
reconstruction of motion, event identity, assignment and resource deltas;
then unroll prior-only for several **decision intervals** with the predicted
ledger fed back, using a growing horizon and event-focused sampling. At
deployment, observe after a short imagined segment or a real decision,
update the posterior, and replan. The simulator is used only to measure
paired action outcomes, never as a hidden transition inside the model.

This follows the distinction between observation updates and prior-only
imagination in DreamerV3 and PlaNet, the short receding-horizon planning
pattern in TD-MPC, and the persistent duration of an option in
Sutton–Precup–Singh. It does not inherit their empirical performance
guarantees: our exogenous scheduler, contention graph and route-control
duration differ. A future autonomous test must generate its own next
controller actions and task schedule or explicitly condition on a stated
external schedule. Replaying factual future route controls against a
diverged predicted state is only a conditional dynamics diagnostic.

At seed-3 frame 900, evaluating just two route candidates for three 30-Hz
frames took **4.60 s** in FP32 on the current server. H100 BF16 dynamics with
FP32 terminal-value scoring reduced that measured call to **1.27 s**, while
preserving the route ranking. The alternative's predicted completion score
exceeded keeping the route by only **0.0004 tasks**, versus
the value model's 2.72-task held-out MAE, so the planner abstained. Across
seed 3, the 1,203 eligible route commits occurred in 629 of 9,000 frames;
the median gap between control-bearing frames was 9.5 frames (0.32 s).
This implementation is therefore still too slow for per-decision online
replanning on this workload. Each latency was one measured call, not a
benchmark distribution; candidate batching and cached graph features still
need work. BF16 for the value head itself rounded the two scores to the same
number, so the value head remains FP32.
