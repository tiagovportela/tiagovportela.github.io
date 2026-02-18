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

---

## The Problem: Sequential Bottlenecks in a Multi-Stage Pipeline

A typical pose estimation pipeline involves several distinct stages:

1. **Frame acquisition** — read a frame from the camera sensor
2. **Person detection** — run a lightweight model to locate a bounding box
3. **Landmark estimation** — run a heavier model to predict body keypoints within the bounding box
4. **Visualisation** — draw the skeleton overlay and write the output frame

In the sequential design, these stages execute one after another on a single thread. The CPU is never doing more than one thing at a time. While stage 3 (landmark estimation) is running inference, stage 1 (the camera) is idle — even though it could already be reading the next frame. This is the core inefficiency: **idle time caused by serialisation**.


![Sequential Pipeline](/posts/cv-pipeline/sequenctial-pipeline.png)
![Sequential Pipeline threads](/posts/cv-pipeline/sequantial.png)

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

![Pipeline threads](/posts/cv-pipeline/multithread-pipeline.png)
![Pipeline threads](/posts/cv-pipeline/multithread.png)

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

![Thread lifecycle](/posts/cv-pipeline/lifecycle.png)

The `ExecutorService` with a fixed thread pool of four threads manages lifecycle. After submitting all tasks, `shutdown()` followed by `awaitTermination()` ensures the main thread waits for the pipeline to drain completely.

---

## Comparison: Sequential vs. Concurrent

### Side-by-Side Architecture

![Design comparison](/posts/cv-pipeline/design-comparation.png)

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

| | Frames Processed | Frames Dropped |
|---|---|---|
| Single-Thread | 499 / 500 (100%) | 0 (0%) |
| Multi-Thread  | 180 / 500 (36%) | 320 (64%) |

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