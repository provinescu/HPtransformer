# Complete Unified Classic Musical RT Guide

## HPtransformerRT V30.3.0 and HPTransformerStudio V8.7.3

**8D | Classic Presets | Live Concert | OSC | ControlBus | Sessions | Stage Mode**

> **Philosophy:** Classic Musical RT prioritizes responsiveness, contrast, diversity, and low CPU usage. Advanced RT processing is bypassed in the engine, and Advanced RT names have been removed from the Classic Studio.

## Contents

1. Overview
2. Installation
3. Structural 8 x 8 configuration
4. Startup
5. Classic Musical RT architecture
6. Learning and generation
7. General Classic presets
8. Classic Live Concert
9. Complete Classic sets
10. 8D OSC
11. 8D ControlBus
12. Global and per-dimension direct tracking
13. Stage Mode
14. Runtime parameters and Studio pages
15. Memory, trajectory, surprise, and replay
16. AutoTune, MetaLearn, and RCU
17. Sessions, presets, and restoration
18. Diagnostics and comparison protocol
19. Compatibility and migration
20. Complete examples
21. Troubleshooting
22. Recommended concert configuration

## 1. Overview

This edition aligns the engine, Studio, preset names, and documentation around a direct musical profile with eight inputs and eight outputs. Studio V8.7.3 adds **Classic Live Concert**, retains Stage Mode, and no longer displays an Advanced RT page.

| Element | Version | Role |
|---|---:|---|
| `HPtransformerRT` | 30.3.0 | Classic 8D learning, prediction, and generation. |
| `HPTransformerStudio` | 8.7.3 | Interface, OSC, ControlBus, presets, sessions, and Stage Mode. |
| Default profile | 8D | Eight inputs and eight outputs. |
| Stage preset | Classic Live Concert | Balance of responsiveness, variety, and stability. |
| Minimal preset | Classic Ultra CPU | Strong adaptation at minimal cost. |

> **Essential difference:** the Classic branch does more than switch Advanced RT OFF. Its main path bypasses normalization, multiresolution history, and stability snapshots.

## 2. Installation

1. Remove older definitions of `HPtransformerRT` and `HPTransformerStudio`.
2. Install `HPtransformerRT_V30_3_0_ClassicMusicalRT_Default8x8.sc` in `Classes`.
3. Install `HPTransformerStudio_V8_7_3_ClassicMusicalRT_LiveConcert_HarmonizedPresets_SceneMode_Default8x8.sc` in `Classes`.
4. Install `HPtransformerRT.schelp` and `HPTransformerStudio.schelp` in `HelpSource/Classes`.
5. Recompile the Class Library.
6. Rebuild the SCDoc index if required:

```supercollider
SCDoc.indexAllDocuments(true);
```

> Never compile two files containing the same class name at the same time. Keep previous versions outside the SuperCollider `Extensions` folder.

## 3. Structural 8 x 8 Configuration

| Parameter | Default | Type |
|---|---:|---|
| `inputSize` | 8 | Structural |
| `outputSize` | 8 | Structural |
| `windowSize` | 8 | Structural |
| `longMemorySize` | 16 | Structural |
| `hiddenSize` | 32 | Structural |
| `numHeads` | 4 | Structural |
| `headSize` | 8 | Structural |
| `numExperts` | 4 | Structural |

Structural parameters cannot be changed while the instance is running. To use another dimensionality, create a new instance and keep `outputSize <= inputSize`.

## 4. Startup

### 4.1 Minimal Creation

```supercollider
(
~hpTR = HPtransformerRT.new;
~studio = HPTransformerStudio.new(~hpTR, \hpTR).front;
)
```

### 4.2 Equivalent Explicit Creation

```supercollider
~hpTR = HPtransformerRT.new(
    inputSize: 8,
    outputSize: 8,
    windowSize: 8,
    longMemorySize: 16,
    hiddenSize: 32,
    numHeads: 4,
    headSize: 8,
    numExperts: 4
);
```

### 4.3 Ultra CPU Without Studio

```supercollider
(
~hpTR = HPtransformerRT.new;
~hpTR.setClassicUltraCPUFastLearn;
)
```

## 5. Classic Musical RT Architecture

| Block | Function |
|---|---|
| Direct cleaning | Bounds and validates the vector without Advanced RT conditioning. |
| Multi-head attention | Analyzes short-term temporal context. |
| Mixture of experts | Combines specialized transformations. |
| Long memory | Stores and recalls selected events. |
| Trajectory memory | Uses velocity and acceleration when recall is enabled. |
| Replay | Relearns previous events according to `replayRate`. |
| Protection | Limits interference and the rate of weight modification. |
| RCU | Optional dual runtime, disabled in the main Classic presets. |

The Classic path is intentionally shorter:

```text
input -> cleaning -> Transformer -> generation
```

## 6. Learning and Generation

### 6.1 `learnEvent`

`learnEvent` learns temporal relationships between events. In Classic Ultra CPU, the application or Studio decimates calls to reduce cost.

```supercollider
~x = [0.20, 0.40, 0.70, 0.50, 0.30, 0.80, 0.60, 0.10];
~hpTR.learnEvent(~x);
```

### 6.2 `generateStep`

`generateStep` returns the engine's raw output. Studio direct tracking is applied separately, after generation.

```supercollider
~y = ~hpTR.generateStep(~x);
```

### 6.3 Decimation Without Studio

```supercollider
(
~counter = 0;
~process = { |input|
    ~counter = ~counter + 1;
    if((~counter % 4) == 0, {
        ~hpTR.learnEvent(input)
    });
    ~hpTR.generateStep(input);
};
)
```

## 7. General Classic Presets

| Preset | Character | Worker / Learn-N |
|---|---|---|
| Classic - Cautious | Slow learning, strong protection | 20 Hz / 8 |
| Classic - Balanced | General-purpose compromise | 30 Hz / 4 |
| Classic - Responsive | Fast adaptation and mobile generation | 40 Hz / 3 |
| Classic - Light Continuity | Moderate classic memory and trajectory | 25 Hz / 4 |
| Classic - Maximum Stability | Strong replay and protection | 15 Hz / 8 |
| Classic Musical Direct | Lightweight musical baseline | 20 Hz / 8 |
| Classic Live Concert | Stage balance | 20 Hz / 6 |
| Classic Generation Only | External learning OFF | 20 Hz / 16 |
| Classic Ultra CPU | Strong adaptation, minimal cost | 20 Hz / 4 |

## 8. Classic Live Concert

`Classic Live Concert` is recommended for monitored performance. It retains light replay, memory recall, and trajectory recall without excessive CPU load.

```text
learningRate: 0.00030
gradientClip: 0.45
surpriseThreshold: 0.055
surpriseGain: 1.35
adaptationFastRate: 1.55
adaptationSlowRate: 0.38
replayRate: 0.03
memoryRecallSize: 1
trajectoryRecallSize: 1
generationWindowSize: 4
attentionTemperature: 1.10
deltaScale: 0.34
trajectoryExplorationGain: 0.004
```

| Operational setting | Value |
|---|---|
| OSC processing | 20 Hz |
| Learn/N | 6 |
| Learning | ON |
| Generation | ON |
| OSC details | OFF |
| RCU | OFF |
| AutoTune | OFF |
| MetaLearn | OFF |

> **Stage advice:** start with Direct Tracking = 0. After evaluating the model, add very light 8D tracking only if selected dimensions must remain linked to the gesture.

## 9. Complete Classic Sets

| Set | Description | 8D tracking |
|---|---|---|
| Classic Cautious | Slow learning and output close to the gesture | `0.20, 0.10, 0.05, 0.20, 0.10, 0.05, 0.20, 0.10` |
| Classic Balanced | General-purpose compromise | `0.25, 0.12, 0.05, 0.25, 0.12, 0.05, 0.25, 0.12` |
| Classic Expressive | Greater generative freedom | `0.10, 0.05, 0.02, 0.10, 0.05, 0.02, 0.10, 0.05` |
| Classic Installation | Contained learning and generation | `0.30, 0.15, 0.08, 0.30, 0.15, 0.08, 0.30, 0.15` |

## 10. 8D OSC

The callback checks `inputSize`. With the default profile, exactly eight values must follow the OSC address.

```supercollider
n = NetAddr("127.0.0.1", 57120);
n.sendMsg(
    "/hptransformer/input",
    0.20, 0.40, 0.70, 0.50,
    0.30, 0.80, 0.60, 0.10
);
```

| Mode | Learning | Output |
|---|---|---|
| Learn + Generate | According to Learn/N | `generateStep`, then tracking |
| Learn only | According to Learn/N | None |
| Predict only | No | `predict` |
| Generate only | No | `generateStep` |

## 11. 8D ControlBus

```supercollider
~bus = Bus.control(s, 8);
~studio.attachControlBus(~bus);
```

The reader Synth must read eight channels. **Direct Bus** can be disabled independently from OSC network output.

## 12. Global and Per-Dimension Direct Tracking

Direct tracking is Studio post-processing applied after `generateStep` and before OSC or ControlBus output.

```text
output[i] = (model[i] * (1.0 - mix[i])) + (input[i] * mix[i])
```

- `0` means model output only.
- `1` means direct input only.
- Intermediate values produce a weighted blend.
- Tracking does not modify model weights.
- Direct tracking is not applied to direct calls to `generateStep`.

Light stage example:

```text
0.12, 0.06, 0.03, 0.12, 0.06, 0.03, 0.12, 0.06
```

## 13. Stage Mode

Stage Mode suspends visual activity without stopping musical processing.

- The periodic refresh routine is no longer called.
- Graphs, heatmaps, monitors, counters, and indicators stop updating.
- OSC Details is forced OFF.
- Visual log collection is suspended.
- OSC, the worker, `learnEvent`, `generateStep`, ControlBus, and network output continue.
- Returning to Control Mode triggers one refresh.

> Stage Mode does not reduce the internal cost of `learnEvent`. To lower that cost, increase Learn/N or lower Processing Hz.

## 14. Runtime Parameters and Studio Pages

| Page | Function |
|---|---|
| Dashboard | Presets, states, reset, and Stage Mode. |
| Controls | Learning, memory, generation, and torus mask. |
| Manual Parameters | Safe runtime parameter editing. |
| Morphing | Progressive interpolation of one parameter. |
| Real-time / Bus / OSC | Worker, Learn/N, tracking, and destinations. |
| Complete Sets | Coordinated configurations. |
| Graphs / Heatmaps | Visual diagnostics. |
| Specialized Presets | Generation, memory, surprise, AutoTune, and MetaLearn. |
| RCU Snapshots | Dual-runtime control. |
| Logs | Logs and export. |

## 15. Memory, Trajectory, Surprise, and Replay

| Parameter | Musical effect | Cost |
|---|---|---|
| `memoryRetrievalGain` | Influence of recalled memories | Increases with recall workload |
| `memoryRecallSize` | Number of memories recalled | High at large values |
| `trajectoryRetrievalGain` | Gestural/trajectory influence | Moderate |
| `trajectoryRecallSize` | Number of trajectories recalled | Increases with value |
| `replayRate` | Frequency of relearning | Significant |
| `protectionStrength` | Slows sensitive modifications | Low |
| `surpriseGain` | Strengthens adaptation to atypical events | Low |

Classic Ultra CPU sets `replayRate`, `memoryRetrievalGain`, and `trajectoryRetrievalGain` to zero. Classic Live Concert retains lightweight versions of these mechanisms.

## 16. AutoTune, MetaLearn, and RCU

- AutoTune automatically adjusts selected generation parameters.
- MetaLearn adjusts selected learning-strategy parameters.
- RCU maintains a publication runtime distinct from the training runtime.
- The main Classic presets disable these functions to reduce cost and improve concert predictability.

## 17. Sessions, Presets, and Restoration

A complete session preserves the engine, `runtimeConfig`, Learning and Generation states, OSC settings, direct tracking, graphs, and logs. After loading, Studio reassigns the configured external variable:

```supercollider
topEnvironment[externalTransformerKey] = transformer;
```

Verification after loading:

```supercollider
[
    \key, ~studio.externalTransformerKey,
    \sameObject, ~hpTR === ~studio.transformer
].postln;
```

## 18. Diagnostics and Comparison Protocol

```supercollider
~hpTR.config.postln;
~hpTR.statusSilent.postln;
```

For valid comparisons:

1. Create a fresh engine before each comparison.
2. Use the same input sequence and duration.
3. Begin with Direct Tracking = 0.
4. Enable Stage Mode.
5. Compare Learn/N = 8, then 6, 4, and 2.
6. Observe CPU, latency, diversity, stability, and playability.
7. Keep the versions with the best musical results.

## 19. Compatibility and Migration

- Class names remain `HPtransformerRT` and `HPTransformerStudio`.
- Do not compile Classic and Advanced branches simultaneously.
- Advanced parameters may remain in old archives, but the Classic path bypasses them.
- Old 3D sessions remain structurally 3D when reloaded.
- For a clean migration, create a new 8D engine and a new session.
- OSC hosts and Synths must be adapted to eight values and eight channels.

## 20. Complete Examples

### 20.1 Live Concert

```supercollider
(
~hpTR = HPtransformerRT.new;
~studio = HPTransformerStudio.new(~hpTR, \hpTR).front;
)
```

In Studio: **Dashboard > General Presets > Classic Live Concert > Apply**, then enable Stage Mode.

### 20.2 Ultra CPU Without Studio

```supercollider
(
~hpTR = HPtransformerRT.new;
~hpTR.setClassicUltraCPUFastLearn;
~counter = 0;
~process = { |x|
    ~counter = ~counter + 1;
    if((~counter % 4) == 0, {
        ~hpTR.learnEvent(x)
    });
    ~hpTR.generateStep(x);
};
)
```

### 20.3 Direct Tracking Outside Studio

```supercollider
~raw = ~hpTR.generateStep(~input);
~mix = [0.12, 0.06, 0.03, 0.12, 0.06, 0.03, 0.12, 0.06];
~out = Array.fill(~raw.size, { |i|
    (
        (~raw[i] * (1.0 - ~mix[i]))
        + (~input[i] * ~mix[i])
    ).clip(0.0, 1.0);
});
```

## 21. Troubleshooting

| Symptom | Action |
|---|---|
| `INPUT: 8 values expected` | Send exactly eight numbers after the OSC address. |
| High CPU | Use Stage Mode, 15 Hz, Learn/N = 8, Details OFF, and RCU OFF. |
| Adaptation too slow | Set Learn/N to 4 or 2. |
| Monotonous generation | Slightly increase temperature, `deltaScale`, or exploration. |
| Unstable generation | Reduce `learningRate` or exploration, and increase Learn/N. |
| Incomplete ControlBus | Create `Bus.control(s, 8)` and adapt the Synth. |
| Duplicate class | Remove older versions from `Extensions`. |
| Help not visible | Check `HelpSource/Classes`, then rebuild SCDoc. |
| `pages[\dashboard]` error | Use Studio V8.7.3 or later and a single backslash. |

## 22. Recommended Concert Configuration

```text
Preset: Classic Live Concert
OSC processing: 20 Hz
Learn/N: 6
Mode: Learn + Generate
Details: OFF
Direct tracking: 0 for the first test
RCU: OFF
AutoTune: OFF
MetaLearn: OFF
Stage Mode: ON after configuration
```

> Before the concert, test the system continuously for at least 20 to 30 minutes using the same OSC routing, Synth, dimensionality, and controller that will be used on stage.

---

`HPtransformerRT V30.3.0 | HPTransformerStudio V8.7.3 | Classic Musical RT`
