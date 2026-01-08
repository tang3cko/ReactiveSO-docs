---
layout: default
title: Observability & Snapshots
parent: Reactive Entity Sets
grand_parent: Guides
nav_order: 7
---

# Observability & Snapshots

---

## Purpose

This page explains the "True Observability" provided by Reactive Entity Sets. It covers the theoretical background based on Set Theory Semantics and practical applications such as snapshots, time travel debugging, and automated testing.

---

## Theoretical Foundation: Set Theoretic Observability

To understand why RES handles data the way it does, we must look at the nature of "Observation" itself.

### Observation is Filtering

In any system, an observer never sees the raw "Truth" directly. Instead, they see a subset of data filtered through a specific perspective.

Let $T$ be the **Total Set** of all data in the system.
Let $S_n$ be an **Observer** (a filter or view).

The data observed by $S_n$ is:

$$ S_n(T) \subseteq T $$

For example:
- $T$: Raw entity data in memory
- $S_{Inspector}$: Data filtered and formatted for the Unity Inspector
- $S_{GameView}$: Data rendered as pixels on screen
- $S_{Logic}$: Data accessed by a specific AI algorithm

### Time as a Function

Data is not static; it changes over time. Therefore, $T$ is a function of time $i$.

$$ \text{Observed State} = S_n(T_i) $$

This equation reveals the essence of **Observability** in game development:

> **True Observability** means having the ability to capture $T_i$ (the total state at moment $i$) deterministically, so that any Observer $S_n$ can be applied to it at any later time to produce the exact same result.

### The Problem with Traditional Architectures

In traditional Object-Oriented Programming (OOP) or Singleton-based architectures, $T_i$ is scattered across the heap, buried in private fields, and entangled with object references.

- **You cannot capture $T_i$**: It's impossible to serialize the entire heap instantly.
- **You cannot replay $S_n(T_i)$**: Without the exact state, bugs cannot be reproduced deterministically.

---

## The RES Solution: Data as "Truth"

Reactive Entity Sets solves this by strictly separating **Data ($T$)** from **Behavior ($S_n$)**.

### 1. Contiguous Memory Layout
Because RES uses `unmanaged struct` constraints, the entire state $T_i$ of 10,000 entities is just a contiguous block of memory (NativeArray).

### 2. Instant Snapshots (Blitting)
Capturing $T_i$ is not a complex serialization process. It is a raw memory copy (MemCpy/Blit).

- **Speed**: Copying 10,000 entities takes < 1ms.
- **Accuracy**: It is a bitwise exact copy of the state.

### 3. Determinism
Because logic is separated from data (Pure Functions), applying logic $S_A$ to state $T_i$ will **always** produce the same result.

---

## Practical Patterns

### 1. Time Travel Debugging

Since $T_i$ can be captured and restored instantly, you can implement a "Seek Bar" for your game's history.

```csharp
// Store history
history.Add(entitySet.CreateSnapshot(Allocator.Persistent));

// Rewind time (0.0ms switching)
public void SeekToFrame(int frame)
{
    // Restoring is just a memory copy.
    // The "Truth" of the world is instantly overwritten.
    entitySet.RestoreSnapshot(history[frame]);
}
```

**Why this is powerful:**
You are not just replaying inputs; you are restoring the **entire memory state**. You can pause, inspect values, and even **intervene** (change state) to see how the future changes (Butterfly Effect).

### 2. Flight Recorder (Bug Reporting)

Keep a ring buffer of the last 60 seconds of snapshots. When an error occurs, dump this buffer to a file.

**Scenario:**
QA reports "The enemy teleported through the wall."

**Traditional approach:**
Try to reproduce it based on a vague description. (Often impossible)

**RES approach:**
Load the Flight Recorder dump. You now have the exact sequence of $T_{i-60s} \dots T_i$ running on your machine. You can step through frame-by-frame and see exactly **why** the physics engine calculated that position.

### 3. Hot Swapping & State Injection

You can save the current state $T_{now}$ to a binary file, modify it externally, and inject it back into the running game.

```csharp
// Save current state
var snapshot = entitySet.CreateSnapshot(Allocator.Temp);
File.WriteAllBytes("state.bin", ToBytes(snapshot));

// ... Edit state.bin in external tool ...

// Inject back
var bytes = File.ReadAllBytes("state.bin");
var newSnapshot = FromBytes(bytes);
entitySet.RestoreSnapshot(newSnapshot);
```

### 4. Parallel Testing with Golden Snapshots

Instead of writing complex setup code for unit tests, you can use Snapshots.

1. **Record**: Run the game, reach a complex state, and save it as a "Golden Snapshot".
2. **Test**: In your test, load the Golden Snapshot into a new `ReactiveEntitySet`.
3. **Verify**: Run your logic and assert the output.

This enables **Data-Driven Tests** that are extremely fast and can run in parallel, because they don't depend on Unity Scenes or GameObjects.

---

## Summary

Observability in RES is not just about seeing values in the Inspector. It is about the **Mathematical Guarantee** that your data is:

1.  **Isolated** (from logic and views)
2.  **Capturable** (as a snapshot)
3.  **Restorable** (deterministically)

By treating your game state as a series of $T_i$, you move from "guessing what happened" to "knowing exactly what happened."

---

## Next steps

- [Snapshot API Reference]({{ '/en/reference/reactive-entity-sets-api' | relative_url }}#snapshot-api)
- [Job System Integration](job-system) - How snapshots enable safe threading
