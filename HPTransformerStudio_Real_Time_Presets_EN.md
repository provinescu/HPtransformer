# HPtransformer Studio: Real-Time Presets

**Starting values for 8D streams**

Apply one preset, listen for **2 to 5 minutes**, and then modify only **one parameter family at a time**.

## Preset Table

| Situation | LR | Att. T | Router T | Delta | Memory | Replay | Protection | Trajectory: retrieval / velocity / acceleration | Diversity: noise / repulsion | Direct Tracking | Effect / Caution |
|---|---:|---:|---:|---:|---:|---:|---:|---|---|---:|---|
| Balanced musician tracking | 0.00032 | 0.88 | 0.95 | 0.27 | 0.14 | 0.08 | 0.18 | 0.11 / 0.065 / 0.010 | 0.00022 / 0.0014 | 0.35 | Versatile. Monitor loss and saturation. |
| Ultra-responsive | 0.00062 | 1.05 | 0.82 | 0.38 | 0.07 | 0.035 | 0.10 | 0.075 / 0.075 / 0.018 | 0.00045 / 0.0020 | 0.55 | Fast attacks. Risk of nervous behavior. |
| Expressive legato | 0.00024 | 0.68 | 0.90 | 0.17 | 0.18 | 0.09 | 0.22 | 0.16 / 0.090 / 0.005 | 0.00004 / 0.00055 | 0.45 | Connected lines. Smoother and slower response. |
| Rhythmic / percussive | 0.00050 | 1.08 | 0.78 | 0.48 | 0.055 | 0.025 | 0.11 | 0.050 / 0.050 / 0.024 | 0.00035 / 0.0018 | 0.25 | Clear attacks. Lower Delta if saturation occurs. |
| Call and response | 0.00030 | 0.86 | 0.92 | 0.29 | 0.19 | 0.10 | 0.20 | 0.115 / 0.060 / 0.009 | 0.00018 / 0.0013 | 0.10 | Autonomous response after listening. |
| Faithful accompaniment | 0.00020 | 0.64 | 0.88 | 0.20 | 0.24 | 0.14 | 0.34 | 0.12 / 0.060 / 0.006 | 0.00004 / 0.00065 | 0.65 | Close to the learned vocabulary. |
| Shared improvisation | 0.00036 | 1.20 | 0.80 | 0.36 | 0.12 | 0.065 | 0.15 | 0.095 / 0.055 / 0.014 | 0.00075 / 0.0026 | 0.20 | More autonomous. Monitor diversity and surprise. |
| Sensors / continuous gesture | 0.00034 | 0.78 | 0.96 | 0.25 | 0.10 | 0.06 | 0.18 | 0.18 / 0.105 / 0.018 | 0.00012 / 0.0009 | 0.40 | Prioritizes velocity and acceleration. |
| Cautious long session | 0.00014 | 0.72 | 0.95 | 0.21 | 0.20 | 0.18 | 0.46 | 0.11 / 0.052 / 0.006 | 0.00008 / 0.00075 | 0.35 | Stable and resistant to forgetting. Slower adaptation. |
| Scene change | 0.00048 | 1.08 | 0.82 | 0.35 | 0.12 | 0.07 | 0.14 | 0.09 / 0.060 / 0.016 | 0.00055 / 0.0021 | 0.25 | Reacts to discontinuities. Stabilize after the transition. |

## Legend

| Abbreviation / Column | Full parameter or meaning |
|---|---|
| LR | `learningRate` |
| Att. T | `attentionTemperature` |
| Router T | `routerTemperature` |
| Memory | `memoryRetrievalGain` |
| Trajectory | retrieval / velocity / acceleration |
| Diversity | noise / repulsion |
| Direct Tracking | Post-processing applied after generation: `0` = model only, `1` = input only. |

## Recommended Shared Settings

```supercollider
(
    gradientClip: 0.60,
    expertScale: 0.22,
    replayBatchSize: 1,
    autoTune: false,
    metaLearn: false
)
```

Keep **AutoTune OFF** and **MetaLearn OFF** during diagnostics so their adaptation does not obscure the effect of the preset under test.

## Diagnostic Adjustments

### If Loss Is Unstable

1. Reduce `learningRate` by **30%**.
2. Set `gradientClip` to **0.40**.
3. Increase `attentionTemperature` by **0.10**.

Worked example for **Balanced musician tracking**:

```text
Original LR:     0.00032
Reduced by 30%:  0.000224
Original Att. T: 0.88
Adjusted Att. T: 0.98
Gradient clip:   0.40
```

### If Output Is Too Repetitive

1. Multiply repulsion by **2**.
2. Then increase exploration slightly if needed.
3. Do not raise all noise parameters simultaneously.

Example for **Faithful accompaniment**:

```text
Repulsion: 0.00065 -> 0.00130
```

### If CPU or OSC Slows Down

Use the following lighter settings:

```supercollider
(
    replayRate: 0.03,
    recallSize: 1, // or 2
    replayBatchSize: 1
)
```

Also keep the SuperCollider Post window quiet during performance or benchmarking.

## Comparison Protocol

1. Use the same input source and routing for every preset.
2. Keep the stream at eight dimensions.
3. Reset or recreate the engine before a controlled comparison when necessary.
4. Apply one preset.
5. Listen for 2 to 5 minutes.
6. Record loss, surprise, diversity, latency, CPU, and perceived playability.
7. Change only one parameter family.
8. Repeat for the same duration.

These presets are starting points, not universal optimum values. They should be compared on the same material for the same duration.
