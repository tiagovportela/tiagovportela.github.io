+++ 
title = "Optimizing Mobile Computer Vision"
date = "2025-07-21"
draft = false
description = "How I optimized a real-time pose estimation pipeline."
tags = ["computer-vision", "mobile", "optimization", "parallel-computing", "multithreading", "producer-consumer"]
categories = ["engineering"]
+++

# From Sequential to Concurrent: How a Pipeline Redesign Doubled Throughput in a Mobile Pose Estimation System

Mobile computer vision is brutal on resources. Every millisecond counts, every CPU cycle is contested, and the gap between a smooth real-time experience and a stuttering one is often just a matter of *how* you sequence your work — not *how much* work you do.

This post documents an investigation into improving the frame rate of a pose estimation pipeline running on a smartphone. The original design was sequential: one frame, one thread, one stage at a time. The new design is pipelined: stages run concurrently on dedicated threads, connected by conflated queues. The results reveal a clear throughput–latency trade-off that is worth understanding carefully before choosing one approach over the other.

The full source code and benchmark scripts are available on GitHub: [ConcurrentPoseInference](https://github.com/tiagovportela/ConcurrentPoseInference).

---

## The Problem: Sequential Bottlenecks in a Multi-Stage Pipeline

A typical pose estimation pipeline involves several distinct stages:

1. **Frame acquisition** — read a frame from the camera sensor
2. **Person detection** — run a lightweight model to locate a bounding box
3. **Landmark estimation** — run a heavier model to predict body keypoints within the bounding box
4. **Visualisation** — draw the skeleton overlay and write the output frame

In the sequential design, these stages execute one after another on a single thread. The CPU is never doing more than one thing at a time. While stage 3 (landmark estimation) is running inference, stage 1 (the camera) is idle — even though it could already be reading the next frame. This is the core inefficiency: **idle time caused by serialisation**.

{{< mermaid >}}
graph LR
    %% Direction and Theme
    direction LR

    %% Subgraphs for logical grouping
    subgraph Input_Stage [Data Acquisition]
        A[/"fa:fa-camera-retro Camera Source"/]
    end

    subgraph Core_Processing [AI Pipeline]
        B("Frame Preprocessor")
        C("Pose Detector")
        D("Pose Tracker")
    end

    subgraph Output_Stage [Result]
        E("Visualizer")
    end

    %% Connection Logic with enhanced labels
    A ==>|Raw Frame| B
    B -->|Input Tensor| C
    C -->|Bounding Box| D
    D ==>|Landmarks| E

    %% Professional Styling
    style A fill:#FFB3FF,stroke:#D400D4,stroke-width:2px,color:#333
    style B fill:#f9f9f9,stroke:#333,stroke-width:2px
    style C fill:#f9f9f9,stroke:#333,stroke-width:2px
    style D fill:#f9f9f9,stroke:#333,stroke-width:2px
    style E fill:#e1f5fe,stroke:#01579b,stroke-width:2px

    %% Subgraph Styling
    style Input_Stage fill:none,stroke:#ccc,stroke-dasharray: 5 5
    style Core_Processing fill:#fcfcfc,stroke:#ccc,stroke-dasharray: 5 5
    style Output_Stage fill:none,stroke:#ccc,stroke-dasharray: 5 5
{{< /mermaid >}}

{{< mermaid >}}
gantt
    title Single Thread Utilization (Synchronous Processing)
    dateFormat  YYYY-MM-DD
    axisFormat  %M:%S
    
    section Core 1 (t1)
    Frame 1         :active, f1, 2024-01-01, 1d
    Detection 1     :active, d1, after f1, 1d
    Landmarks 1     :active, l1, after d1, 1d
    Visualization 1 :active, v1, after l1, 1d

    section Core 2 (t2)
    idle            :crit, 2024-01-01, 4d

    section Core 3 (t3)
    idle            :crit, 2024-01-01, 4d

    section Core 4 (t4)
    idle            :crit, 2024-01-01, 4d
{{< /mermaid >}}

### Amdahl's Law and the Limits of Parallelism

Before reaching for threads, it is worth asking: *how much can parallelism actually help?* Amdahl's Law gives us a theoretical ceiling.

If a fraction `p` of the work can be parallelised and the rest `(1 - p)` must remain sequential, then the maximum speedup achievable with `N` processors is:

$$
Speedup(N) = \frac{1} {((1 - p) + p / N)}
$$

As `N → ∞`, the speedup approaches `1 / (1 - p)`. In other words, if 90% of the pipeline is parallelisable, the best possible speedup is 10×, regardless of how many cores you throw at it.

In our case the pipeline has four stages. If each stage takes roughly equal time and all four can run concurrently, the theoretical maximum speedup is approximately 4×. In practice, synchronisation overhead, memory bandwidth contention, and the sequential fraction (queue management, memory copies) reduce this. The results we measured — a **2.12× throughput gain** — are consistent with a real-world parallel fraction somewhere around 50–60%, which is typical for a pipeline where one stage (landmark inference) dominates the total time budget.

### Pipeline Parallelism vs. Data Parallelism

There are two classic approaches to parallelising a pipeline like this:

- **Data parallelism**: replicate the entire pipeline across multiple threads, each processing a different frame independently. Works well when stages are independent; requires replicating large model weights in memory.
- **Pipeline parallelism**: each stage runs in its own thread; frames flow through stages like an assembly line. Memory is shared; stages overlap in time but no frame is processed by more than one thread simultaneously.

We chose **pipeline parallelism** because the TFLite models are expensive to duplicate and the stages are naturally sequential per frame. The design maps cleanly onto a producer–consumer architecture.

---

## The New Design: A Concurrent Producer–Consumer Pipeline

### Architecture Overview

The new system decomposes the pipeline into four dedicated threads connected by bounded, conflated queues.
{{< mermaid >}}
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Helvetica, Arial, sans-serif', 'fontSize': '14px', 'lineColor': '#5e6c84'}}}%%
flowchart LR
    %% Define Professional Minimalist Styles
    classDef process fill:#ffffff,stroke:#5e6c84,stroke-width:1px,color:#172b4d;
    classDef queue fill:#f4f5f7,stroke:#8993a4,stroke-width:2px,stroke-dasharray: 4 4,color:#172b4d;
    classDef thread fill:#fbfbfc,stroke:#dfe1e6,stroke-width:2px,color:#5e6c84;

    subgraph T1 ["Thread 1: Frame Producer"]
        N1["<strong>CameraSource</strong><br/>read frame"]:::process
    end
    
    %% Changed to stadium shape (horizontal pill)
    Q1(["FrameQueue<br/>(cap=2)"]):::queue
    
    subgraph T2 ["Thread 2: Bounding Box"]
        N2["<strong>PoseDetector</strong><br/>bounding box<br/>inference"]:::process
    end
    
    %% Changed to stadium shape (horizontal pill)
    Q2(["BoundingBoxQueue<br/>(cap=2)"]):::queue
    
    subgraph T3 ["Thread 3: Landmarks Producer"]
        N3["<strong>PoseTracker</strong><br/>landmark<br/>inference"]:::process
    end
    
    %% Changed to stadium shape (horizontal pill)
    Q3(["LandmarksQueue<br/>(cap=2)"]):::queue
    
    subgraph T4 ["Thread 4: Landmarks Visualizer"]
        N4["<strong>PoseVisualizer</strong><br/>draw + save<br/>+ metrics"]:::process
    end

    %% Standardized Flow Connections
    N1 --> Q1
    Q1 --> N2
    N2 --> Q2
    Q2 --> N3
    N3 --> Q3
    Q3 --> N4

    %% Apply Subgraph Styles
    class T1,T2,T3,T4 thread
{{< /mermaid >}}



{{< mermaid >}}
gantt
    title Pipelined Execution (Multithreaded)
    dateFormat  YYYY-MM-DD
    axisFormat  %M:%S
    
    section Thread 1 (t1)
    Frame 1            :active, f1, 2024-01-01, 1d
    Frame 2            :active, f2, 2024-01-02, 1d
    Frame 3            :active, f3, 2024-01-03, 1d
    Intermediate...    :done,   f_gap, 2024-01-04, 3d
    Frame n            :active, fn, 2024-01-07, 1d

    section Thread 2 (t2)
    idle               :crit, i2, 2024-01-01, 1d
    Detection 1        :active, d1, 2024-01-02, 1d
    Detection 2        :active, d2, 2024-01-03, 1d
    Detection 3        :active, d3, 2024-01-04, 1d
    Intermediate...    :done,   d_gap, 2024-01-05, 2d
    Detection n        :active, dn, 2024-01-07, 1d

    section Thread 3 (t3)
    idle               :crit, i3, 2024-01-01, 2d
    Landmark 1         :active, l1, 2024-01-03, 1d
    Landmark 2         :active, l2, 2024-01-04, 1d
    Landmark 3         :active, l3, 2024-01-05, 1d
    Intermediate...    :done,   l_gap, 2024-01-06, 1d
    Landmark n         :active, ln, 2024-01-07, 1d

    section Thread 4 (t4)
    idle               :crit, i4, 2024-01-01, 3d
    Visualization 1    :active, v1, 2024-01-04, 1d
    Visualization 2    :active, v2, 2024-01-05, 1d
    Intermediate...    :done,   l_gap, 2024-01-06, 1d
    Visualization n    :active, vn, 2024-01-07, 1d
{{< /mermaid >}}

Each queue has a capacity of 2. This is intentional: it limits buffering (and therefore latency growth) while still decoupling adjacent stages enough to absorb minor jitter.

### The Conflated Queue: Freshness over Completeness

The central design decision is the **conflated queue** strategy. When the queue is full and a new frame arrives, the oldest frame is discarded rather than blocking the producer:

```java
public void put(FrameData frame) {
    // If the queue is full, drop the oldest frame to make room
    while (!queue.offer(frame)) {
        queue.poll(); // discard stale frame
    }
}
```

This has an important consequence: the downstream consumer always works on the *freshest available frame*, never waiting on stale data. For a real-time application — where you care about what is happening *now* — this is the right trade-off. Dropped frames are preferable to accumulated lag.

The poison-pill pattern propagates shutdown: when the camera source is exhausted, a sentinel `null` frame is inserted into the first queue, which each consumer passes downstream before exiting.

### Thread Lifecycle and Shutdown

{{< mermaid >}}
%%{init: {
  'theme': 'base',
  'themeVariables': {
    'fontFamily': 'Helvetica, Arial, sans-serif',
    'fontSize': '14px',
    'actorBkg': '#e3f2fd',
    'actorBorder': '#1565c0',
    'actorTextColor': '#000000',
    'signalColor': '#000000',
    'signalTextColor': '#000000',
    'loopBkg': '#fff8e1',
    'loopTextColor': '#000000'
  }
}}%%
sequenceDiagram
    participant T1 as FrameProducer (T1)
    participant Q1 as frameQueue
    participant T2 as BoundingBoxConsumer (T2)
    participant Q2 as boundingBoxQueue
    participant T3 as LandmarksProducer (T3)
    participant Q3 as landmarksQueue
    participant T4 as LandmarksConsumer (T4)

    loop for each frame
        T1->>Q1: put(frame)
        Q1->>T2: take() → detect()
        T2->>Q2: put(frameWithBox)
        Q2->>T3: take() → track()
        T3->>Q3: put(frameWithLandmarks)
        Q3->>T4: take() → visualize()
    end

    T1->>Q1: put(POISON_PILL)
    T2->>Q2: put(POISON_PILL)
    T3->>Q3: put(POISON_PILL)
    
    T4-->>T4: exit
{{< /mermaid >}}

The `ExecutorService` with a fixed thread pool of four threads manages lifecycle. After submitting all tasks, `shutdown()` followed by `awaitTermination()` ensures the main thread waits for the pipeline to drain completely.

---

## Comparison: Sequential vs. Concurrent

### Side-by-Side Architecture

{{< mermaid >}}
flowchart TB
    subgraph CONCURRENT["Concurrent — Multi-Thread"]
        direction TB
        T1["Thread 1\nRead Frame"]
        T2["Thread 2\nDetect Person"]
        T3["Thread 3\nEstimate Landmarks"]
        T4["Thread 4\nVisualise & Save"]

        T1 -- "queue A" --> T2
        T2 -- "queue B" --> T3
        T3 -- "queue C" --> T4
    end

    subgraph SEQUENTIAL["Sequential — Single-Thread"]
        direction TB
        S1["Read Frame"]
        S2["Detect Person Bounding Box"]
        S3["Estimate Landmarks"]
        S4["Visualise & Save"]

        S1 --> S2
        S2 --> S3
        S3 --> S4
        S4 -- "next frame" --> S1
    end

    style CONCURRENT fill:#d0e8f8,stroke:#2d6aad,color:#0a2a4a
    style SEQUENTIAL fill:#d0e8f8,stroke:#2d6aad,color:#0a2a4a

    style T1 fill:#1e5799,stroke:#4a9ede,color:#ffffff
    style T2 fill:#1e5799,stroke:#4a9ede,color:#ffffff
    style T3 fill:#1e5799,stroke:#4a9ede,color:#ffffff
    style T4 fill:#1e5799,stroke:#4a9ede,color:#ffffff

    style S1 fill:#155a8a,stroke:#4a9ede,color:#ffffff
    style S2 fill:#155a8a,stroke:#4a9ede,color:#ffffff
    style S3 fill:#155a8a,stroke:#4a9ede,color:#ffffff
    style S4 fill:#155a8a,stroke:#4a9ede,color:#ffffff
{{< /mermaid >}}

The key conceptual difference is that in the sequential design, time advances *vertically* — each stage must complete before the next begins. In the concurrent design, time advances *horizontally* — at any given moment, all four threads are doing useful work on different frames.

### Performance Results

![Performance results](/posts/cv-pipeline/plot_comparation.png)

Both systems were benchmarked on the same 500-frame sequence. The multi-threaded pipeline used the conflated queue, simulating a 30 FPS camera source.

#### Throughput (inter-frame completion interval)

| Metric      | Single-Thread | Multi-Thread |
|-------------|:-------------:|:------------:|
| Mean (ms)   | 192.1         | 91.0         |
| Median (ms) | 191.8         | 90.4         |
| Std (ms)    | 5.8           | 4.9          |
| Min (ms)    | 176.4         | 74.7         |
| Max (ms)    | 227.0         | 109.4        |
| IQR (ms)    | 6.7           | 5.3          |
| **FPS**     | **5.2**       | **11.0**     |

The multi-threaded pipeline delivers **2.12× higher throughput** (11.0 vs 5.2 FPS), with slightly *lower* variance — a notable bonus, since the pipelined design smooths out per-stage jitter across concurrent stages.

#### Latency (wall-clock time per frame)

| Metric      | Single-Thread | Multi-Thread |
|-------------|:-------------:|:------------:|
| Mean (ms)   | 158.7         | 222.7        |
| Median (ms) | 158.6         | 223.1        |
| Std (ms)    | 5.8           | 11.1         |
| Min (ms)    | 143.1         | 196.3        |
| Max (ms)    | 193.5         | 251.7        |
| IQR (ms)    | 6.9           | 17.5         |

Latency is **40% higher** in the concurrent pipeline (223 vs 159 ms median). This is expected: a frame now waits in multiple queues as it travels through four stages, accumulating queuing delay at each boundary. The single-threaded version has zero queuing overhead — latency equals pure inference time.

The statistical significance is unambiguous. A Mann–Whitney U test yields `p ≈ 3.4×10⁻⁸⁸` for both metrics, confirming the differences are not noise.

#### Frame Processing Coverage

The most striking difference is **frame coverage**:

![Frame coverage](/posts/cv-pipeline/frame_drop.png)

The conflated queue dropped 64% of frames. From a raw completeness perspective this looks alarming, but in the context of a *real-time* application it is the intended behaviour: the system prioritised processing the most recent frames rather than accumulating a backlog. At 30 FPS input and ~11 FPS processing capacity, dropping is mathematically inevitable — the camera produces frames roughly 2.7× faster than the pipeline can consume them.

---

## Interpreting the Trade-Off

The results expose a fundamental tension in real-time pipeline design:

- **Throughput** measures how many frames are *completed per unit time*. Higher throughput means the output is more up-to-date on average.
- **Latency** measures how long it takes for a *single frame* to travel end-to-end. Lower latency means the system responds faster to events.

Pipeline parallelism improves throughput at the cost of latency. This is always true: a frame must wait at each queue boundary for the downstream stage to become free. The more stages, the more queuing delay.

| Concern | Best Choice | Reason |
|---|---|---|
| Batch / offline processing | Single-threaded | 100% frame coverage, lower latency per frame, simpler to reason about |
| Real-time display / feedback | Multi-threaded | Higher FPS output, always processes the latest frame |
| Latency-critical applications (e.g. gesture control) | Single-threaded or hybrid | Multi-threaded latency may introduce perceptible delay |
| Throughput-critical applications (e.g. activity monitoring) | Multi-threaded | 2× more frames processed per second |

---


## Conclusion

A relatively modest architectural change — wrapping four sequential stages in dedicated threads connected by conflated queues — more than doubled the sustained frame rate of the pose estimation pipeline from 5.2 to 11.0 FPS. This required no changes to the inference models, no hardware upgrades, and no data parallelism.

The cost is real: latency increased by 40%, and the conflated queue discards roughly two-thirds of incoming frames under a 30 FPS camera load. For a smartphone application displaying live skeleton overlays, the throughput gain outweighs both costs — the user sees smoother output, and the system always reacts to the *current* frame rather than one that arrived several hundred milliseconds ago.

The deeper lesson is architectural: **concurrency is not free, but neither is serialisation.** Understanding exactly where your pipeline spends its time — and which stages can overlap — is the prerequisite for any meaningful performance improvement.

## Notes

One simplification in this study is that we run the full person detector on every processed frame. In a real-world deployment, this is rarely necessary.

Adjacent video frames are highly correlated. A more efficient approach would be to run the expensive detector only initially (or periodically) to find the subject, and then use a lightweight tracker (like optical flow or a Kalman filter) to follow the bounding box in subsequent frames. Re-running the detector would only be triggered if tracking fails or at a fixed low frequency (e.g. 1 Hz). This "detect-and-track" strategy could theoretically double or triple the effective frame rate by skipping the heavy detection inference step for the majority of frames.