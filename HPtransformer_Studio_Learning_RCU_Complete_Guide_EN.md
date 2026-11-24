# HPtransformer Studio: Complete Learning and RCU Guide

**HPtransformer & HPTransformer Studio Pro**  
SuperCollider 3.14 | Final edition with Learning, RCU, morphing, graphs, and heatmaps

> **Edition:** October 21, 2026  
> Adds a technically corrected explanation of Learning ON/OFF and RCU. The compact interface targets 13-inch displays and uses `ScrollView`, a Morphing menu, EMA graphs, and resized heatmaps.

## Quick Start

```supercollider
~hp = HPtransformer.new;
~studio = HPTransformerStudio.new(~hp).front;
```

1. Apply **Studio Balanced**.
2. Check `TorusMask`.
3. Enable Learning when appropriate for the session.
4. Enable RCU when training and inference must be separated.
5. Inspect the graphs before publishing a snapshot.

## 1. Architecture

| Component | Role |
|---|---|
| `HPtransformer` | Learning, memory, prediction, generation, and RCU. |
| `HPTransformerStudio` | GUI control, OSC, morphing, graphs, heatmaps, and sessions. |

## 2. Parameters and Saved Files

```supercollider
residualScale = 0.34;
replayRate = 0.12;
temperature = 1.0;
```

| Format | Contents |
|---|---|
| `.hptpreset` | Runtime parameters. |
| `.hptset` | Parameters, `TorusMask`, and states. |
| `.hptsession` | Engine, GUI, OSC, graphs, and logs. |
| `.csv` | Graph histories. |
| `.png` | Heatmaps. |

## 3. Morphing

| Scope | Method |
|---|---|
| Shared | `morphParameter` |
| RCU runtime | `morphRuntimeParameter` |
| Training | `morphTrainingParameter` |

The Morphing drop-down list is generated from `runtimeParameterNames` and retains numeric values only. When a parameter is selected, the target starts from its current value.

## 4. Graphs and Heatmaps

- Raw curves use light colors; EMA trends use darker colors.
- Loss scales: Auto, 0 to 0.10, and 0 to 1.00.
- Other metric scales: Auto, 0 to 1.00, and 0 to 2.00.
- Graph rendering uses `graphDrawScale = 0.82`.
- Heatmap rendering uses `heatDrawScale = 0.82`.
- Data collection rejects non-finite values and does not inject artificial zeros.

## 5. Learning ON/OFF and RCU Activity

Learning and RCU control two independent dimensions:

- **Learning** determines whether the engine may modify its knowledge.
- **RCU** determines whether a training version is isolated from the published version used for inference.
- **Generation** determines whether the engine may produce generative output.

| Control | Question it answers |
|---|---|
| Learning ON/OFF | May the model modify weights, optimizer state, and memories? |
| RCU ON/OFF | Are updates isolated before they are published to inference? |
| Generation ON/OFF | May the engine generate output? |

> **Important:** RCU ON does not imply Learning ON. Learning ON does not imply RCU ON. The states are independent.

### 5.1 Learning ON

```supercollider
transformer.enableLearning;
```

Learning ON authorizes learning paths such as `learnEvent` and `learn`. Depending on the engine configuration, updates may affect weights, optimizer state, memory, replay, surprise processing, and loss metrics.

Recommended uses:

- calibration and continuous learning;
- external OSC learning when **External Learning** is also enabled;
- sessions in which LOSS, SURPRISE, MEMORY RECALL, and TRAJECTORY RECALL can be monitored.

### 5.2 Learning OFF

```supercollider
transformer.disableLearning;
```

Learning OFF disables updates performed by the learning path. `predict` and `generateStep` may continue when their corresponding modes remain enabled.

Adaptive mechanisms that are separate from Learning, including AutoTune, MetaLearn, or some memory-writing mechanisms, must be disabled separately if they do not depend on the Learning state.

## 6. RCU OFF: Direct Shared Model

```supercollider
transformer.disableUnifiedRCU;
```

With RCU disabled, the explicit Prepare, Commit, and Publish cycle is not used. Learning effects may be applied directly to the active path or synchronized automatically, depending on the `HPtransformer` implementation.

| State | Consequence |
|---|---|
| Learning ON + RCU OFF | Learning updates affect the active path immediately, or are synchronized automatically. |
| Learning OFF + RCU OFF | The learning path is disabled; prediction and generation remain available according to their own states. |

**Advantage:** simple, responsive behavior.  
**Risk:** a poor update can immediately affect generation.  
**Typical use:** development, rapid tests, and adaptive improvisation.

## 7. RCU ON: Training Separated from Inference

```supercollider
transformer.enableUnifiedRCU;
```

With RCU active, snapshots are designed to separate a changing training state from a stable published state. Depending on the implementation, a snapshot may include weights, runtime parameters, memories, or only a subset of them.

```text
Events -> Training runtime -> Prepare -> Commit -> Publish -> Published runtime
```

## 8. The RCU Cycle

| Command | Role |
|---|---|
| `prepareSnapshot` | Builds a candidate version from the training state. |
| `commitSnapshot` | Validates the prepared snapshot. |
| `publishNow` | Makes the validated version active for inference. |
| `discardPendingSnapshot` | Drops the pending candidate without changing the published runtime. |

### 8.1 Prepare

```supercollider
transformer.prepareSnapshot;
```

Creates a candidate version without necessarily changing the published version.

### 8.2 Commit

```supercollider
transformer.commitSnapshot;
```

Validates the candidate before publication.

### 8.3 Publish

```supercollider
transformer.publishNow;
```

Makes the validated candidate available to the inference runtime.

### 8.4 Discard

```supercollider
transformer.discardPendingSnapshot;
```

Deletes the candidate while leaving the published runtime unchanged.

## 9. The Four Learning/RCU Combinations

| Learning | RCU | Behavior | Typical use |
|---|---|---|---|
| OFF | OFF | Learning path disabled. | Diagnostics, prediction, stable generation. |
| ON | OFF | Direct update of the active model. | Development and immediate adaptation. |
| OFF | ON | Stable published runtime, no new learning. | Performance and distribution. |
| ON | ON | Separate training with controlled publication. | Safe continuous real-time learning. |

> For a public-facing system that must keep learning without immediately changing its output, use **Learning ON + RCU ON**, then run Prepare, Commit, and Publish only after checking the metrics.

## 10. Relationship with OSC

On the OSC page, **External Learning ON/OFF** is an additional authorization. OSC data can train the model only when:

1. the selected OSC mode allows learning;
2. `oscLearning` is enabled; and
3. Learning is enabled in the engine.

| OSC mode | External learning | Output |
|---|---|---|
| Learn + Generate | Yes, if authorized | `generateStep` |
| Learn only | Yes, if authorized | None |
| Predict only | No | `predict` |
| Generate only | No | `generateStep` |

```supercollider
n = NetAddr("127.0.0.1", 57120);
n.sendMsg(
    "/hptransformer/input",
    0.1, 0.2, 0.3, 0.4,
    0.5, 0.6, 0.7, 0.8
);
```

## 11. Relationship with Morphing

| Morphing scope | Conceptual target | Example |
|---|---|---|
| Shared | General setting | `temperature`: 1.0 -> 1.4 |
| RCU runtime | Active inference | `deltaScale`: 0.20 -> 0.40 |
| Training | Learning model | `learningRate`: 0.00035 -> 0.00010 |

RCU Runtime morphing changes an inference-runtime parameter without forcing the same transition on the training runtime. The exact result depends on the separation implemented by `morphRuntimeParameter`.

Training morphing targets the learning model according to the implementation of `morphTrainingParameter`.

## 12. Recommended Configurations

| Scenario | Learning | Generation | RCU | Recommendation |
|---|---:|---:|---:|---|
| Initial calibration | ON | ON | OFF | Keep AutoTune and MetaLearn OFF to isolate effects. |
| Controlled learning | ON | ON | ON | Observe, Prepare, Commit, Publish. |
| Stable performance | OFF | ON | ON | Use the published version. |
| Pure diagnostics | OFF | ON | OFF | Reduce competing mechanisms. |

### 12.1 Cautious Publication Procedure

1. Enable Learning and RCU.
2. Allow the training runtime to evolve.
3. Inspect LOSS, SURPRISE, MEMORY RECALL, and TRAJECTORY RECALL.
4. Run Prepare.
5. Run Commit if the metrics are acceptable.
6. Run Publish at the chosen musical or operational moment.
7. Use Discard if the candidate is not satisfactory.

## 13. Diagnostics

| Symptom | Check or action |
|---|---|
| The model does not learn | Confirm Learning ON, a learning-capable OSC mode, and External Learning ON. |
| Generation changes too quickly | Enable RCU and publish manually. |
| The published version does not change | Run Prepare, Commit, then Publish. |
| A candidate version is poor | Run `discardPendingSnapshot`. |
| Runtime morphing has no expected effect | Check RCU and the selected morphing scope. |
| Learning is OFF but output remains active | This is normal when Generation remains ON. |

## 14. Operational Summary

```text
Learning ON  = authorizes learning
Learning OFF = freezes learned knowledge
RCU ON       = separates training from published inference
RCU OFF      = direct path through the active model
Prepare      = creates a candidate version
Commit       = validates the candidate
Publish      = makes the candidate active
Discard      = abandons the candidate
```

## 15. Learning/RCU Flow

The following diagram summarizes the corrected flow. Learning controls updates to the training runtime. RCU keeps a stable published version until the Prepare, Commit, and Publish cycle is completed.

![Corrected Learning and RCU flow](assets_learning_rcu/media/image2.png)

**Figure 1.** Separation of learning, candidate version, and published runtime.

> **Correct reading:** incoming events split into two paths. The inference path reaches the published runtime directly. The Learning path is used only when Learning is enabled.

### 15.1 Corrected Technical Legend

| Marker | Element | Meaning | Consequence |
|---|---|---|---|
| Light green | Incoming events | Data received by the engine. | May feed inference independently of Learning. |
| Yellow diamond | Learning active? | Decision to allow the learning path. | Yes: training runtime; No: no update through this path. |
| Pink | Training runtime | Working state that may evolve. | Exact contents depend on `HPtransformer`. |
| Cyan | Prepare / Commit | Prepares and validates a candidate. | Must not be presented as published before Publish. |
| Blue | Publish / published runtime | Version used for inference. | Becomes active after successful publication. |
| Green | Prediction / Generation | Output computation. | May continue with Learning OFF if enabled. |
| Red | Discard | Candidate cancellation. | Published runtime remains unchanged. |
| Gray | Learning OFF | Disables the learning path. | Does not necessarily stop all other adaptive mechanisms. |

## 16. Studio Screenshots

The screenshots below illustrate the compact Studio layout. The source screenshot is low resolution and was enlarged for print in the original guide.

![Graphs page overview](assets_learning_rcu/media/image3.png)

**Figure 2.** General view of the Graphs page in `HPTransformerStudio`.

### 16.1 Navigation and Top Controls

![Navigation controls](assets_learning_rcu/media/image4.png)

**Figure 3.** Navigation among Dashboard, Controls, Morphing, Graphs, Heatmaps, and RCU.

### 16.2 Operational Area

![Metric graphs](assets_learning_rcu/media/image5.png)

**Figure 4.** Five time-series metrics: Loss, Surprise, Entropy, Memory Recall, and Trajectory Recall.

For a future documentation release, use separate native-resolution captures of Dashboard, Graphs, Heatmaps, Morphing, and RCU Snapshots.
