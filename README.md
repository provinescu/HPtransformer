# HPtransformer

**A real-time musical transformer for SuperCollider 3.14**

HPtransformer is an experimental neural system written in SuperCollider for learning transitions between normalized multidimensional musical events and generating new trajectories in real time.

It combines multi-head attention, a mixture of experts, associative long-term memory, trajectory memory, prioritized replay, adaptive generation controls, and targeted Read-Copy-Update (RCU) snapshots.

> **Project status:** HPtransformer is intended for artistic research, algorithmic composition, live coding, gestural interaction, and generative music systems.

---

## Table of contents

- [Main features](#main-features)
- [Requirements](#requirements)
- [Installation](#installation)
- [Quick start](#quick-start)
- [Core workflow](#core-workflow)
- [Architecture](#architecture)
- [Input and output dimensions](#input-and-output-dimensions)
- [Learning](#learning)
- [Prediction and generation](#prediction-and-generation)
- [Targeted RCU snapshots](#targeted-rcu-snapshots)
- [Memory systems](#memory-systems)
- [AutoTune](#autotune)
- [MetaLearn](#metalearn)
- [Real-time parameters](#real-time-parameters)
- [Musical presets](#musical-presets)
- [Continuous real-time example](#continuous-real-time-example)
- [Mapping the output to a SynthDef](#mapping-the-output-to-a-synthdef)
- [Diagnostics](#diagnostics)
- [Saving and restoring](#saving-and-restoring)
- [Performance notes](#performance-notes)
- [Troubleshooting](#troubleshooting)

---

## Main features

- Online learning from normalized musical event vectors
- Multi-head attention with feature gates
- Mixture-of-experts routing
- Long-term associative memory
- Short-term temporal memory
- Dynamic trajectory memory with velocity and acceleration
- Prioritized replay
- Adam optimization with gradient clipping
- Torus-aware dimensions for cyclic parameters
- Autoregressive generation
- Diversity, drift, and local-contraction correction
- AutoTune for generation parameters
- MetaLearn for learning-strategy parameters
- Targeted inference snapshots without full-object `deepCopy`
- Unified RCU architecture with active and pending runtimes
- Shared, training-only, and runtime-only parameter scopes
- Parameter morphing
- Symmetric and asymmetric dimensions, for example `8 → 8` or `6 → 3`
- Native SuperCollider help support through `HPtransformer.schelp`

---

## Requirements

- **SuperCollider 3.14**
- A working SuperCollider language installation (`sclang`)
- No external UGen is required by the transformer class itself
- macOS, Linux, or Windows should be possible wherever SuperCollider 3.14 runs, but each platform should be tested independently

HPtransformer runs in the SuperCollider language process. It does not directly generate audio. Its output vectors must be mapped to Synth parameters, MIDI, OSC, Patterns, or other musical processes.

---

## Installation

Recommended extension layout:

```text
HPtransformer/
├── Classes/
│   └── HPtransformer.sc
├── HelpSource/
│   └── Classes/
│       └── HPtransformer.schelp
├── Examples/
│   ├── QuickStart.scd
│   ├── AsymmetricIO.scd
│   └── RealtimeGeneration.scd
└── README.md
```

Place the `HPtransformer` folder inside:

```supercollider
Platform.userExtensionDir;
```

Typical macOS location:

```text
~/Library/Application Support/SuperCollider/Extensions/
```

Then select:

```text
Language > Recompile Class Library
```

Make sure that only one file defines the `HPtransformer` class. Duplicate class definitions will prevent the SuperCollider class library from compiling.

To open the native help page:

```supercollider
HelpBrowser.openHelpFor("Classes/HPtransformer");
```

If necessary, rebuild the help index:

```supercollider
SCDoc.indexAllDocuments(true);
```

---

## Quick start

```supercollider
(
t = HPtransformer.new(
    inputSize: 8,
    outputSize: 8
);

t.enableUnifiedRCU;

~eventA = [
    0.10, 0.20, 0.30, 0.40,
    0.50, 0.60, 0.70, 0.80
];

~eventB = [
    0.12, 0.24, 0.31, 0.43,
    0.52, 0.62, 0.74, 0.81
];

// The first event initializes the sequence.
t.learnEvent(~eventA);

// The next event learns a transition.
~loss = t.learnEvent(~eventB);

// Publish the learned inference state to the generator.
t.publishNow;

// Generate one step.
~seed = Array.fill(8, 0.5);
~seed = t.generateStep(~seed);

[
    \loss, ~loss,
    \output, ~seed
].postln;
)
```

---

## Core workflow

The normal workflow is:

```text
Learn → Prepare snapshot → Commit snapshot → Generate
```

In SuperCollider:

```supercollider
t.learnEvent(event);
t.prepareSnapshot;
t.commitSnapshot;
seed = t.generateStep(seed);
```

Or, for manual tests:

```supercollider
t.learnEvent(event);
t.publishNow;
seed = t.generateStep(seed);
```

`publishNow` is a convenience method that calls `prepareSnapshot` and then `commitSnapshot`.

Do not publish after every event unless the model is small and the measured snapshot cost is acceptable. A practical starting point is one publication every **5–10 seconds** or every **32–64 learned events**.

---

## Architecture

```text
Input events
     │
     │ learnEvent / learnPair
     ▼
Main training model
├── trainable weights
├── Adam state
├── replay memory
├── long-term memory
├── trajectory memory
└── MetaLearn
     │
     │ exportInferenceSnapshot
     ▼
Pending runtime
     │
     │ commitSnapshot
     ▼
Active runtime
├── predict
├── generateStep
├── diversity control
├── drift correction
└── AutoTune
     │
     ▼
Generated output vectors
```

The main object learns. The active runtime generates. The pending runtime receives the next targeted snapshot.

### Why RCU?

RCU means **Read-Copy-Update**:

1. The generator **reads** a stable active runtime.
2. The training model **copies** only the inference state into the pending runtime.
3. `commitSnapshot` **updates** the active runtime by swapping references.

This controls exactly when new learning becomes audible or visible in the generated output.

---

## Input and output dimensions

HPtransformer supports:

```text
outputSize <= inputSize
```

Examples:

```text
8 → 8
6 → 3
12 → 6
```

A configuration such as `3 → 6` is not supported by the current delta-based output architecture.

### Symmetric dimensions

With identical dimensions, the current output becomes the next input, preserving the original autoregressive behavior:

```supercollider
t = HPtransformer.new(
    inputSize: 8,
    outputSize: 8
);
```

### Asymmetric dimensions

A `6 → 3` model can use six contextual features to predict three musical controls:

```supercollider
(
t = HPtransformer.new(
    inputSize: 6,
    outputSize: 3
);

t.enableUnifiedRCU;

~input = [
    0.10, 0.20, 0.30,
    0.40, 0.50, 0.60
];

~target = [
    0.70,
    0.35,
    0.85
];

~loss = t.learnPair(~input, ~target);
t.publishNow;

~prediction = t.predict(~input);
~prediction.postln; // Three values
)
```

### Asymmetric autoregressive feedback

By default, the output replaces the first output dimensions of the previous input. Remaining context dimensions are preserved:

```text
Previous input: [0.20, 0.30, 0.40, 0.50, 0.60, 0.70]
Output:         [0.72, 0.43, 0.81]
Next input:     [0.72, 0.43, 0.81, 0.50, 0.60, 0.70]
```

Custom feedback is available through `generateMapped`:

```supercollider
(
~feedback = { |output, previousInput|
    [
        output[0],
        output[1],
        output[2],
        (previousInput[3] + 0.01).wrap(0.0, 1.0),
        ((previousInput[4] * 0.995) + 0.0025).clip(0.0, 1.0),
        (previousInput[5] + 0.005.rand2).clip(0.0, 1.0)
    ]
};

~sequence = t.generateMapped(
    [0.20, 0.30, 0.40, 0.50, 0.60, 0.70],
    64,
    ~feedback
);
)
```

---

## Learning

### Sequential event learning

```supercollider
t.learnEvent(event);
```

The first event initializes the sequence. Each later event learns the transition from the previous event.

### Explicit supervised learning

```supercollider
~loss = t.learnPair(input, target);
```

This is the recommended method when `inputSize != outputSize`.

### Learning configuration

```supercollider
(
t.setTrainingParameters((
    learningRate: 0.00035,
    replayRate: 0.08,
    replayBatchSize: 1,
    protectionStrength: 0.16,
    interferenceTestSteps: 0
), false);
)
```

Start with a low event rate. `0.005.wait` does not limit learning to 5 ms: SuperCollider completes the full synchronous call first and only then begins the wait.

---

## Prediction and generation

### Prediction

```supercollider
~prediction = t.predict(input);
```

The result always contains `outputSize` values.

### One autoregressive step

```supercollider
~output = t.generateStep(input);
```

### Sequence generation

```supercollider
~sequence = t.generate(seed, 32);
```

### Custom feedback generation

```supercollider
~sequence = t.generateMapped(seed, 64, feedbackFunction);
```

### Manual feedback step

```supercollider
~output = t.generateStep(~currentInput);
~currentInput = t.outputToNextInput(
    ~output,
    ~currentInput
);
```

---

## Targeted RCU snapshots

### Enable RCU

```supercollider
t.enableUnifiedRCU;
```

### Prepare a version

```supercollider
t.prepareSnapshot;
```

This copies only the inference state:

- forward weights and biases
- attention weights
- feature gates
- router and expert weights
- long-term memory
- trajectory memory
- inference parameters

It does not copy the full optimizer and training history.

### Commit a version

```supercollider
t.commitSnapshot;
```

The active generation state is transferred to the pending runtime, then the active and pending runtimes are swapped.

### Inspect RCU state

```supercollider
t.unifiedRCUStatus.postln;
```

Useful values:

```text
trainingVersion
publishedVersion
snapshotPending
learningEnabled
generationEnabled
```

### Disable RCU

```supercollider
t.disableUnifiedRCU;
```

This removes the internal runtimes but preserves the main training model. Direct prediction and generation then use the main model.

---

## Memory systems

### Short-term memory

A recent temporal window used by multi-head attention.

### Long-term associative memory

Events are selected according to surprise and novelty. Retrieval combines similarity, distance, importance, and usage.

Important parameters:

```text
memoryWriteThreshold
memoryRetrievalGain
memoryRetrievalTemperature
memoryRecallSize
memoryDecay
memoryUsageDecay
```

### Prioritized replay

Replay stores input-target pairs and reinforces difficult or important transitions.

```text
replayRate
replayBatchSize
replayPriorityMix
replayUniformMix
```

### Trajectory memory

Trajectory memory stores input, velocity, acceleration, importance, age, and usage.

Important parameters:

```text
trajectoryRecallSize
trajectoryRetrievalGain
trajectoryVelocityGain
trajectoryAccelerationGain
trajectoryExplorationGain
```

---

## AutoTune

AutoTune adapts generation parameters from measured novelty and diversity.

It controls:

```text
temperature
trajectoryExplorationGain
diversityNoiseGain
```

Enable and inspect:

```supercollider
t.enableAutoTune;
t.autoTuneStatus.postln;
```

Balanced starting configuration:

```supercollider
(
t.setParameters((
    autoTuneInterval: 8,
    autoTuneStrength: 0.20,
    autoTuneSmoothing: 0.90,
    autoTuneTargetNovelty: 0.18,
    autoTuneTargetDiversity: 0.12,
    autoTuneExplorationMin: 0.0005,
    autoTuneExplorationMax: 0.0300,
    autoTuneNoiseMin: 0.0,
    autoTuneNoiseMax: 0.0100,
    autoTuneTemperatureMin: 0.50,
    autoTuneTemperatureMax: 2.50
), false);

t.resetAutoTune;
t.enableAutoTune;
)
```

Disable AutoTune before manually morphing parameters that AutoTune controls.

---

## MetaLearn

MetaLearn adapts the learning strategy from error, surprise, recall, and interference measurements.

It controls:

```text
learningRate
replayRate
protectionStrength
memoryRetrievalGain
```

Enable and inspect:

```supercollider
t.enableMetaLearning;
t.metaLearnStatus.postln;
```

Balanced starting configuration:

```supercollider
(
t.setParameters((
    metaLearnInterval: 16,
    metaLearnStrength: 0.12,
    metaLearnSmoothing: 0.95,
    metaLearnTargetError: 0.015,
    metaLearnTargetSurprise: 0.08,
    metaLearnTargetRecall: 0.50,
    metaLearnTargetInterference: 0.0
), false);

t.resetMetaLearning;
t.enableMetaLearning;
)
```

Disable MetaLearn before manually morphing the parameters that MetaLearn controls.

---

## Real-time parameters

### Shared and persistent

```supercollider
t.setParameter(\deltaScale, 0.35, false);
```

The value is applied to the training model and both runtimes.

### Training only

```supercollider
t.setTrainingParameter(
    \learningRate,
    0.00025,
    false
);
```

### Active runtime only

```supercollider
t.setRuntimeParameter(
    \temperature,
    1.25,
    false
);
```

A runtime-only value may be replaced by the next snapshot.

### Multiple parameters

```supercollider
(
t.setParameters((
    temperature: 1.0,
    deltaScale: 0.30,
    memoryRetrievalGain: 0.10,
    trajectoryExplorationGain: 0.0045,
    diversityNoiseGain: 0.00035,
    driftEmaDecay: 0.985,
    driftGain: 0.45,
    driftMaxCorrection: 0.0025
), false);
)
```

### Parameter morphing

```supercollider
t.morphParameter(
    \temperature,
    1.35,
    8.0,
    160
);
```

Other scopes:

```supercollider
t.morphRuntimeParameter(...);
t.morphTrainingParameter(...);
t.stopMorph(\temperature);
t.stopAllMorphs;
```

---

## Musical presets

### Balanced

```supercollider
(
t.setParameters((
    temperature: 1.0,
    deltaScale: 0.30,
    memoryRetrievalGain: 0.10,
    trajectoryRetrievalGain: 0.07,
    trajectoryVelocityGain: 0.045,
    trajectoryExplorationGain: 0.0045,
    diversityNoiseGain: 0.00035,
    localDiversityGain: 0.055
), false);
)
```

### Fluid and legato

```supercollider
(
t.setParameters((
    temperature: 0.82,
    deltaScale: 0.18,
    memoryRetrievalGain: 0.16,
    memoryRetrievalTemperature: 0.22,
    memoryRecallSize: 2,
    trajectoryVelocityGain: 0.060,
    trajectoryExplorationGain: 0.0012,
    diversityNoiseGain: 0.00008
), false);
)
```

### Controlled exploration

```supercollider
(
t.setParameters((
    temperature: 1.35,
    deltaScale: 0.40,
    memoryRetrievalGain: 0.08,
    trajectoryExplorationGain: 0.0090,
    diversityRepulsionGain: 0.0028,
    diversityNoiseGain: 0.00085,
    localDiversityGain: 0.075
), false);
)
```

### Faithful to learned material

```supercollider
(
t.setParameters((
    temperature: 0.72,
    deltaScale: 0.24,
    memoryRetrievalGain: 0.22,
    memoryRetrievalTemperature: 0.14,
    memoryRecallSize: 1,
    trajectoryExplorationGain: 0.0007,
    diversityNoiseGain: 0.00002
), false);
)
```

---

## Continuous real-time example

```supercollider
(
~running = false;
Tdef(\hpMain).stop.clear;

t = HPtransformer.new(
    inputSize: 8,
    outputSize: 8
);

t.enableUnifiedRCU;

~running = true;
~phase = 0.0;
~seed = Array.fill(8, 0.5);
~counter = 0;

Tdef(\hpMain, {
    while({ ~running == true }, {
        var event;

        event = Array.fill(8, { |index|
            (
                (~phase + (index * 0.37)).sin
                * 0.45
                + 0.5
            ).clip(0.0, 1.0)
        });

        t.learnEvent(event);
        ~seed = t.generateStep(~seed);

        ~phase = ~phase + 0.035;
        ~counter = ~counter + 1;

        if((~counter % 50) == 0, {
            t.publishNow;
        });

        0.1.wait;
    });
}).play(SystemClock);
)
```

Stop:

```supercollider
(
~running = false;

t.disableLearning;
t.disableGeneration;
t.stopAllMorphs;

Tdef(\hpMain).stop.clear;
)
```

SuperCollider Routines are cooperative. A stop request cannot interrupt a synchronous method that is already executing. Measure `learnEvent` and `prepareSnapshot`, and use a reasonable cadence.

---

## Mapping the output to a SynthDef

Example mapping from an 8-dimensional output:

```supercollider
(
~playVector = { |vector|
    var x;

    x = vector.collect({ |value|
        value.asFloat.clip(0.0, 1.0)
    });

    Synth(\oberheimInspired, [
        \freq, x[0].linexp(0.0, 1.0, 55.0, 1760.0),
        \amp, x[1].linlin(0.0, 1.0, 0.03, 0.16),
        \cutoff, x[2].linexp(0.0, 1.0, 150.0, 9000.0),
        \resonance, x[3].linlin(0.0, 1.0, 0.05, 0.72),
        \sawPulseMix, x[4],
        \detune, x[5].linlin(0.0, 1.0, 0.02, 0.40),
        \ampAttack, x[6].linexp(0.0, 1.0, 0.003, 1.50),
        \ampRelease, x[7].linexp(0.0, 1.0, 0.08, 4.00)
    ]);
};
)
```

Always verify the expected vector length before accessing dimensions.

---

## Diagnostics

```supercollider
t.unifiedRCUStatus.postln;
t.status;
t.unifiedRuntimeStatus;
t.autoTuneStatus.postln;
t.metaLearnStatus.postln;
```

Memory diagnostics:

```supercollider
t.memoryTest(input).postln;
t.trajectoryMemoryTest(input).postln;
```

Measure synchronous operations:

```supercollider
(
~start = Main.elapsedTime;
t.prepareSnapshot;
[
    \prepareDuration,
    Main.elapsedTime - ~start
].postln;
)
```

---

## Saving and restoring

```supercollider
t.saveArchive(
    "~/Documents/HPtransformer.archive".standardizePath
);
```

After restoring an object, recreate or enable the RCU runtimes before generation:

```supercollider
~restored.enableUnifiedRCU;
```

> Note: if the historical `readArchive` type check still references an older class name, update that check to `HPtransformer` in the class source.

---

## Performance notes

HPtransformer performs substantial synchronous computation in `sclang`.

Recommended starting points:

- Learning interval: **0.10 s**
- Generation interval: **0.05–0.15 s**
- Snapshot publication: every **5–10 s** or every **32–64 events**
- `replayBatchSize`: **1** during initial testing
- `interferenceTestSteps`: **0** for real-time operation unless explicitly testing interference
- Avoid posting every generated vector to the Post Window

If audio events report `late`, increase server latency, simplify the model, reduce event frequency, or move heavy publication away from time-critical boundaries.

---

## Troubleshooting

### `Class already defined`

More than one `.sc` file defines `HPtransformer`. Remove or move old copies outside the SuperCollider extension search paths.

### `Message ... not understood`

Recompile the class library and create a new instance. Existing instances do not gain newly added methods.

### Generation does not reflect recent learning

Publish a new snapshot:

```supercollider
t.publishNow;
```

### A runtime-only parameter disappears

Use `setParameter`, not `setRuntimeParameter`, if the value must survive the next snapshot.

### Generation is too repetitive

Increase gradually:

```text
trajectoryExplorationGain
diversityNoiseGain
diversityRepulsionGain
localDiversityGain
```

### Generation is too unstable

Reduce gradually:

```text
deltaScale
trajectoryExplorationGain
diversityNoiseGain
trajectoryAccelerationGain
```

### Stop commands respond slowly

Measure synchronous calls and reduce learning or snapshot frequency. `Cmd + .` cannot interrupt a long synchronous language operation until control returns to the interpreter.

### Help page shows “Missing documentation”

Verify the exact path and filename:

```text
HelpSource/Classes/HPtransformer.schelp
```

Then rebuild the index:

```supercollider
SCDoc.indexAllDocuments(true);
```

---
