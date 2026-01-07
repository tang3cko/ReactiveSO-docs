---
layout: default
title: Job System Integration
parent: Reactive Entity Sets
grand_parent: Guides
nav_order: 6
---

# Job System Integration

---

## Purpose

This guide explains how to use ReactiveEntitySet with Unity's Job System for high-performance parallel processing. You will learn when to use the Orchestrator pattern, how double buffering works, and how to implement efficient simulations with thousands of entities.

---

## When to use job system integration

### Use the Orchestrator when

- You have **1,000+ entities** updating every frame
- You need **Burst compilation** for performance
- You're running **physics simulations** or **AI calculations**
- You want to **parallelize** entity updates across CPU cores

### Use direct SetData when

- You have **fewer than 1,000 entities**
- Updates are **infrequent** (not every frame)
- You need **immediate** event notifications
- Simplicity is more important than performance

### Performance comparison

| Approach | 1,000 entities | 10,000 entities | 100,000 entities |
|----------|----------------|-----------------|------------------|
| Direct SetData | ~0.5ms | ~5ms | ~50ms |
| Orchestrator + Jobs | ~0.1ms | ~0.3ms | ~2ms |

{: .note }
> These numbers are approximate and depend on your data structure size and update logic complexity.

---

## Core concepts

### Double buffering

The Orchestrator maintains two copies of your data:

```
Frame N:
┌─────────────────┐     ┌─────────────────┐
│  Front Buffer   │     │  Back Buffer    │
│  (Current State)│     │  (Next State)   │
│                 │     │                 │
│  Read by:       │     │  Written by:    │
│  - Rendering    │     │  - Jobs         │
│  - UI           │     │  - Simulation   │
└─────────────────┘     └─────────────────┘

Frame N+1 (after CompleteAndApply):
┌─────────────────┐     ┌─────────────────┐
│  Back Buffer    │     │  Front Buffer   │
│  (Now Current)  │     │  (Now Next)     │
│                 │     │                 │
│  Buffers swap   │ ←→  │  with O(1)      │
│  instantly      │     │  pointer swap   │
└─────────────────┘     └─────────────────┘
```

This pattern allows:

- **Lock-free reads**: Main thread reads front buffer while Jobs write to back buffer
- **No copying**: Buffers swap by pointer exchange, not data copy
- **Consistent state**: Readers always see a complete, consistent frame

### The Orchestrator lifecycle

```csharp
void Start()
{
    // 1. Create Orchestrator (wraps your EntitySet)
    orchestrator = new ReactiveEntitySetOrchestrator<MyData>(entitySet);
}

void Update()
{
    // 2. Schedule Jobs that write to back buffer
    var handle = ScheduleMyJob();
    orchestrator.ScheduleUpdate(handle, entityCount);
}

void LateUpdate()
{
    // 3. Complete Jobs and swap buffers
    orchestrator.CompleteAndApply();
}

void OnDestroy()
{
    // 4. Clean up native memory
    orchestrator?.Dispose();
}
```

---

## Step-by-step implementation

### Step 1: Define your data struct

Your data must be `unmanaged` (no managed references) for Job System compatibility.

```csharp
// Good: All fields are unmanaged types
public struct UnitState
{
    public float2 position;
    public float2 velocity;
    public int health;
    public int teamId;
}

// Bad: Contains managed reference
public struct BadState
{
    public string name;      // string is managed - won't compile
    public GameObject target; // reference type - won't compile
}
```

{: .warning }
> The `unmanaged` constraint is enforced by `ReactiveEntitySetSO<TData>`. If your struct contains managed types, you'll get a compile error.

### Step 2: Create your EntitySet asset

```csharp
using Tang3cko.ReactiveSO;
using UnityEngine;

[CreateAssetMenu(
    fileName = "UnitStateSet",
    menuName = "Reactive SO/Entity Sets/Unit State"
)]
public class UnitStateSetSO : ReactiveEntitySetSO<UnitState>
{
    // Base class provides all functionality
}
```

### Step 3: Create the simulation manager

```csharp
using UnityEngine;
using Unity.Collections;
using Unity.Jobs;
using Tang3cko.ReactiveSO;

public class SimulationManager : MonoBehaviour
{
    [SerializeField] private UnitStateSetSO unitSet;

    private ReactiveEntitySetOrchestrator<UnitState> orchestrator;

    private void Start()
    {
        // Create Orchestrator to manage double buffering
        orchestrator = new ReactiveEntitySetOrchestrator<UnitState>(unitSet);

        // Spawn initial units
        SpawnUnits();
    }

    private void Update()
    {
        if (unitSet.Count == 0) return;

        // Schedule simulation Job
        JobHandle handle = ScheduleSimulation();

        // Tell Orchestrator about the pending Job
        orchestrator.ScheduleUpdate(handle, unitSet.Count);
    }

    private void LateUpdate()
    {
        // Complete Job and swap buffers
        // This makes the new state visible to the EntitySet
        orchestrator?.CompleteAndApply();
    }

    private void OnDestroy()
    {
        // Always dispose to free native memory
        orchestrator?.Dispose();
        orchestrator = null;
    }

    private void SpawnUnits()
    {
        for (int i = 0; i < 10000; i++)
        {
            unitSet.Register(i, new UnitState
            {
                position = Random.insideUnitCircle * 100f,
                velocity = Random.insideUnitCircle,
                health = 100,
                teamId = i % 4
            });
        }
    }

    private JobHandle ScheduleSimulation()
    {
        // Read from front buffer (current state)
        NativeSlice<UnitState> srcData = unitSet.Data;
        NativeSlice<int> srcIds = unitSet.EntityIds;

        // Write to back buffer (next state)
        NativeArray<UnitState> dstData = orchestrator.GetBackBuffer();
        NativeArray<int> dstIds = orchestrator.GetBackBufferIds();

        // Copy IDs (they don't change in this simulation)
        new NativeSlice<int>(dstIds, 0, srcIds.Length).CopyFrom(srcIds);

        // Schedule the simulation Job
        var job = new UnitSimulationJob
        {
            Input = srcData,
            Output = dstData,
            DeltaTime = Time.deltaTime
        };

        return job.Schedule(srcData.Length, 64);
    }
}
```

### Step 4: Write the Burst-compiled Job

```csharp
using Unity.Burst;
using Unity.Collections;
using Unity.Jobs;
using Unity.Mathematics;

[BurstCompile]
public struct UnitSimulationJob : IJobParallelFor
{
    [ReadOnly] public NativeSlice<UnitState> Input;
    public NativeArray<UnitState> Output;
    public float DeltaTime;

    public void Execute(int index)
    {
        UnitState unit = Input[index];

        // Update position based on velocity
        unit.position += unit.velocity * DeltaTime;

        // Boundary wrapping
        unit.position = math.fmod(unit.position + 100f, 200f) - 100f;

        // Write to output buffer
        Output[index] = unit;
    }
}
```

---

## Advanced patterns

### Pattern 1: Reading map data in Jobs

Jobs can read from other NativeArrays alongside the entity data.

```csharp
[BurstCompile]
public struct TerrainAwareJob : IJobParallelFor
{
    [ReadOnly] public NativeSlice<UnitState> Units;
    [ReadOnly] public NativeArray<int> TerrainMap;
    public NativeArray<UnitState> Results;
    public int MapWidth;
    public int MapHeight;

    public void Execute(int index)
    {
        var unit = Units[index];

        // Sample terrain at unit position
        int x = (int)math.clamp(unit.position.x, 0, MapWidth - 1);
        int y = (int)math.clamp(unit.position.y, 0, MapHeight - 1);
        int terrain = TerrainMap[y * MapWidth + x];

        // Modify behavior based on terrain
        if (terrain == 0) // Water
        {
            unit.velocity *= 0.5f; // Slow down
        }

        Results[index] = unit;
    }
}
```

### Pattern 2: Writing to shared data (with caution)

Use `[NativeDisableParallelForRestriction]` when multiple Jobs need to write to shared data.

```csharp
[BurstCompile]
public struct TerritoryPaintJob : IJobParallelFor
{
    [ReadOnly] public NativeSlice<UnitState> Units;
    public NativeArray<UnitState> Results;

    [NativeDisableParallelForRestriction]
    public NativeArray<int> TerritoryMap; // Shared write target

    public int MapWidth;

    public void Execute(int index)
    {
        var unit = Units[index];

        // Paint territory at unit position
        int x = (int)unit.position.x;
        int y = (int)unit.position.y;
        int mapIndex = y * MapWidth + x;

        // Note: Multiple units may write to same cell
        // Last write wins (acceptable for territory painting)
        TerritoryMap[mapIndex] = unit.teamId;

        Results[index] = unit;
    }
}
```

{: .warning }
> When multiple threads write to the same memory location, results are non-deterministic. Only use this pattern when "last write wins" behavior is acceptable.

### Pattern 3: Chaining multiple Jobs

```csharp
private JobHandle ScheduleSimulation()
{
    var srcData = unitSet.Data;
    var dstData = orchestrator.GetBackBuffer();

    // Job 1: Physics update
    var physicsJob = new PhysicsJob
    {
        Input = srcData,
        Output = dstData,
        DeltaTime = Time.deltaTime
    };
    JobHandle physicsHandle = physicsJob.Schedule(srcData.Length, 64);

    // Job 2: AI update (depends on physics)
    var aiJob = new AIJob
    {
        Data = dstData, // Read from physics output
        Targets = targetPositions
    };
    JobHandle aiHandle = aiJob.Schedule(srcData.Length, 64, physicsHandle);

    // Job 3: Collision detection (depends on AI)
    var collisionJob = new CollisionJob
    {
        Data = dstData
    };
    JobHandle collisionHandle = collisionJob.Schedule(srcData.Length, 64, aiHandle);

    return collisionHandle;
}
```

---

## Snapshot API

The Snapshot API allows you to capture and restore the complete state of an EntitySet.

### Creating snapshots

```csharp
// Capture current state
EntitySetSnapshot<UnitState> snapshot = unitSet.CreateSnapshot(Allocator.Persistent);

// snapshot.Data contains copy of all entity data
// snapshot.EntityIds contains copy of all entity IDs
// snapshot.Count contains the entity count
```

### Restoring snapshots

```csharp
// Restore to a previous state
unitSet.RestoreSnapshot(snapshot);

// This:
// 1. Clears the current EntitySet
// 2. Registers all entities from the snapshot
// 3. Fires OnSetChanged event
```

### Memory management

```csharp
// Snapshots allocated with Persistent must be disposed
snapshot.Dispose();

// Or use Allocator.Temp for short-lived snapshots
using (var tempSnapshot = unitSet.CreateSnapshot(Allocator.Temp))
{
    // Use snapshot...
} // Automatically disposed
```

### Use case: Time-travel / Rewind

```csharp
public class TimeController : MonoBehaviour
{
    [SerializeField] private UnitStateSetSO unitSet;

    private List<EntitySetSnapshot<UnitState>> history = new();
    private int maxHistory = 300; // 5 seconds at 60fps

    private void LateUpdate()
    {
        // Record history every frame
        if (history.Count >= maxHistory)
        {
            history[0].Dispose();
            history.RemoveAt(0);
        }
        history.Add(unitSet.CreateSnapshot(Allocator.Persistent));
    }

    public void Rewind(int frames)
    {
        int targetIndex = history.Count - 1 - frames;
        if (targetIndex >= 0 && targetIndex < history.Count)
        {
            unitSet.RestoreSnapshot(history[targetIndex]);
        }
    }

    private void OnDestroy()
    {
        foreach (var snapshot in history)
        {
            snapshot.Dispose();
        }
        history.Clear();
    }
}
```

### Use case: Save/Load

```csharp
public void SaveGame(string path)
{
    var snapshot = unitSet.CreateSnapshot(Allocator.Temp);

    // Convert to byte array for serialization
    byte[] dataBytes = new byte[snapshot.Count * UnsafeUtility.SizeOf<UnitState>()];
    byte[] idBytes = new byte[snapshot.Count * sizeof(int)];

    snapshot.Data.Slice(0, snapshot.Count).CopyTo(
        NativeArrayUnsafeUtility.ConvertExistingDataToNativeArray<UnitState>(
            dataBytes, snapshot.Count, Allocator.None));

    // Save to file...

    snapshot.Dispose();
}
```

---

## Performance tips

### 1. Batch size tuning

The second parameter of `Schedule()` is the batch size. Tune it based on your workload.

```csharp
// Small, fast operations: larger batches
job.Schedule(count, 256);

// Complex operations: smaller batches
job.Schedule(count, 32);

// Rule of thumb: start with 64 and profile
job.Schedule(count, 64);
```

### 2. Minimize main thread work

```csharp
// Bad: Waiting for Job on main thread
void Update()
{
    var handle = ScheduleJob();
    handle.Complete(); // Blocks main thread!
    ProcessResults();
}

// Good: Let Job run while main thread does other work
void Update()
{
    var handle = ScheduleJob();
    orchestrator.ScheduleUpdate(handle, count);
    // Main thread continues with rendering, input, etc.
}

void LateUpdate()
{
    orchestrator.CompleteAndApply(); // Complete at end of frame
}
```

### 3. Avoid allocations in Jobs

```csharp
// Bad: Creating NativeArray inside Job
public void Execute(int index)
{
    var temp = new NativeArray<float>(10, Allocator.Temp); // Allocation!
}

// Good: Pre-allocate and pass as parameter
[NativeDisableParallelForRestriction]
public NativeArray<float> TempBuffer; // Pre-allocated
```

### 4. Use Burst

Always add `[BurstCompile]` to your Jobs for 10-100x performance improvement.

```csharp
[BurstCompile]
public struct MyJob : IJobParallelFor
{
    // Burst compiles this to highly optimized native code
}
```

---

## Common pitfalls

### 1. Forgetting to dispose the Orchestrator

```csharp
// Memory leak!
void OnDestroy()
{
    // orchestrator.Dispose(); // Forgot this!
}

// Correct
void OnDestroy()
{
    orchestrator?.Dispose();
    orchestrator = null;
}
```

### 2. Reading from EntitySet during Job execution

```csharp
// Dangerous: Reading while Job is running
void Update()
{
    orchestrator.ScheduleUpdate(handle, count);

    // This reads front buffer, which is safe
    var state = unitSet[someId]; // OK

    // But modifying triggers events that might cause issues
    unitSet[someId] = newState; // Avoid during Job execution
}
```

### 3. Forgetting to copy IDs

```csharp
// Bug: IDs not copied to back buffer
private JobHandle ScheduleSimulation()
{
    var dstData = orchestrator.GetBackBuffer();
    var dstIds = orchestrator.GetBackBufferIds();

    // Forgot to copy IDs!
    // new NativeSlice<int>(dstIds, 0, srcIds.Length).CopyFrom(srcIds);

    var job = new MyJob { Output = dstData };
    return job.Schedule(count, 64);
}
```

### 4. Using wrong allocator for snapshots

```csharp
// Bug: Temp allocator used for long-lived snapshot
var snapshot = unitSet.CreateSnapshot(Allocator.Temp);
history.Add(snapshot); // Will become invalid after 4 frames!

// Correct: Use Persistent for long-lived data
var snapshot = unitSet.CreateSnapshot(Allocator.Persistent);
history.Add(snapshot);
// Remember to Dispose when done!
```

---

## Complete example

For a complete working example, see the **TinyHistoryDemo** sample in the package:

```
Assets/_Project/Samples/TinyHistoryDemo/
├── Scripts/
│   ├── SimulationManager.cs    // Orchestrator usage
│   └── MapManager.cs           // Texture rendering
└── Scenes/
    └── TinyHistoryDemo.unity
```

This demo simulates 10,000 units painting territory on a map, achieving 200+ fps on modern hardware.

---

## References

- [Job System Integration (Technical)]({{ '/en/design-philosophy/reactive-entity-sets/job-system-integration' | relative_url }}) - Detailed architecture
- [Unity Job System Manual](https://docs.unity3d.com/Manual/JobSystem.html)
- [Burst Compiler Manual](https://docs.unity3d.com/Packages/com.unity.burst@latest)
- [NativeArray Documentation](https://docs.unity3d.com/ScriptReference/Unity.Collections.NativeArray_1.html)
