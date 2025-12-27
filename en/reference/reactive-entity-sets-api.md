---
layout: default
title: Reactive entity sets API
parent: Reference
nav_order: 4
---

# Reactive entity sets API

## Purpose

This reference documents the complete API for ReactiveEntitySetSO and ReactiveEntity. You will find the methods, properties, and events for managing per-entity state.

---

## Overview

Reactive Entity Sets provide three API styles.

| API Style | Class | Use Case |
|-----------|-------|----------|
| ID-based | `ReactiveEntitySetSO<TData>` | Direct ID operations, network sync |
| MonoBehaviour-based | `ReactiveEntitySetSO<TData>` | Convenience wrappers using owner references |
| Entity base class | `ReactiveEntity<TData>` | Auto-registration pattern |

---

## ReactiveEntitySetSO\<TData\>

The core ScriptableObject that stores entity state using a Sparse Set data structure.

### Type constraint

```csharp
public abstract class ReactiveEntitySetSO<TData> : ReactiveEntitySetSO
    where TData : struct
```

TData must be a value type (struct).

---

## Properties

| Property | Type | Description |
|----------|------|-------------|
| Count | `int` | Number of registered entities |
| Data | `ArraySegment<TData>` | Read-only access to all state data |
| EntityIds | `ArraySegment<int>` | Read-only access to all entity IDs |

### Events

| Property | Type | Description |
|----------|------|-------------|
| OnItemAdded | `IntEventChannelSO` | Raised when entity registered (passes ID) |
| OnItemRemoved | `IntEventChannelSO` | Raised when entity unregistered (passes ID) |
| OnDataChanged | `IntEventChannelSO` | Raised when entity data changes (passes ID) |
| OnSetChanged | `VoidEventChannelSO` | Raised on any change |

---

## ID-based API

Use this API for direct control with integer IDs.

### Register

```csharp
public void Register(int id, TData initialData)
```

Registers an entity with its initial state.

```csharp
entitySet.Register(123, new EnemyState { Health = 100 });
```

### Unregister

```csharp
public void Unregister(int id)
```

Removes an entity from the set.

```csharp
entitySet.Unregister(123);
```

### Indexer

```csharp
public TData this[int id] { get; set; }
```

Gets or sets state data. Setting triggers OnDataChanged if the value differs.

```csharp
// Get
EnemyState state = entitySet[123];

// Set
entitySet[123] = new EnemyState { Health = 50 };
```

### GetData

```csharp
public TData GetData(int id)
```

Gets state data. Throws `KeyNotFoundException` if not registered.

```csharp
EnemyState state = entitySet.GetData(123);
```

### TryGetData

```csharp
public bool TryGetData(int id, out TData data)
```

Attempts to get state data without throwing.

```csharp
if (entitySet.TryGetData(123, out var state))
{
    Debug.Log($"Health: {state.Health}");
}
```

### SetData

```csharp
public void SetData(int id, TData data)
```

Sets state data. Raises OnDataChanged if the value differs.

```csharp
entitySet.SetData(123, new EnemyState { Health = 50 });
```

### GetDataRef

```csharp
public ref TData GetDataRef(int id)
```

Gets a reference to state data for direct modification. More efficient for frequent updates but requires manual notification.

```csharp
ref var state = ref entitySet.GetDataRef(123);
state.Health -= damage;
entitySet.NotifyDataChanged(123);  // Must call manually
```

### UpdateData

```csharp
public void UpdateData(int id, Func<TData, TData> updater)
```

Updates state using a transformation function. Automatically triggers OnDataChanged.

```csharp
entitySet.UpdateData(123, state => {
    state.Health -= damage;
    return state;
});
```

### Contains

```csharp
public bool Contains(int id)
```

Checks if an ID is registered.

```csharp
if (entitySet.Contains(123))
{
    // Entity exists
}
```

### NotifyDataChanged

```csharp
public void NotifyDataChanged(int id)
```

Manually triggers data changed event. Use after GetDataRef modifications.

---

## MonoBehaviour-based API

Convenience wrappers that use `owner.GetInstanceID()` as the entity ID.

### Register

```csharp
public void Register(MonoBehaviour owner, TData initialData)
```

Registers using the MonoBehaviour's instance ID.

```csharp
entitySet.Register(this, new EnemyState { Health = 100 });
```

### Unregister

```csharp
public void Unregister(MonoBehaviour owner)
```

Unregisters using the MonoBehaviour reference.

```csharp
entitySet.Unregister(this);
```

### Indexer

```csharp
public TData this[MonoBehaviour owner] { get; set; }
```

Gets or sets state by owner reference.

```csharp
// Get
EnemyState state = entitySet[this];

// Set
entitySet[this] = new EnemyState { Health = 50 };
```

### Other methods

All ID-based methods have MonoBehaviour overloads.

- `GetData(MonoBehaviour owner)`
- `TryGetData(MonoBehaviour owner, out TData data)`
- `SetData(MonoBehaviour owner, TData data)`
- `GetDataRef(MonoBehaviour owner)`
- `UpdateData(MonoBehaviour owner, Func<TData, TData> updater)`
- `Contains(MonoBehaviour owner)`
- `NotifyDataChanged(MonoBehaviour owner)`

---

## Iteration

### ForEach

```csharp
public void ForEach(Action<int, TData> callback)
```

Iterates over all entities. Uses backward iteration for safe modification.

```csharp
entitySet.ForEach((id, state) => {
    if (state.Health <= 0)
    {
        entitySet.Unregister(id);  // Safe during iteration
    }
});
```

### Data property

For performance-critical code, access the underlying array directly.

```csharp
var data = entitySet.Data;
for (int i = 0; i < data.Count; i++)
{
    ProcessState(data.Array[data.Offset + i]);
}
```

---

## Per-entity subscriptions

Subscribe to changes for a specific entity.

### SubscribeToEntity

```csharp
public void SubscribeToEntity(int id, Action<TData, TData> callback)
```

Subscribes to state changes for a specific entity.

```csharp
entitySet.SubscribeToEntity(123, (oldState, newState) => {
    if (oldState.Health != newState.Health)
    {
        OnHealthChanged(newState.Health);
    }
});
```

### UnsubscribeFromEntity

```csharp
public void UnsubscribeFromEntity(int id, Action<TData, TData> callback)
```

Removes a subscription.

---

## Utility methods

### Clear

```csharp
public void Clear()
```

Removes all entities from the set.

### CleanupDestroyed

```csharp
public void CleanupDestroyed()
```

Removes entities whose MonoBehaviour owners have been destroyed. Only affects entities registered with MonoBehaviour owners.

```csharp
// Call periodically if Unregister might not be called reliably
entitySet.CleanupDestroyed();
```

---

## ReactiveEntity\<TData\>

Base class for entities that auto-register to a ReactiveEntitySetSO.

### Abstract members

| Member | Type | Description |
|--------|------|-------------|
| Set | `ReactiveEntitySetSO<TData>` | The set to register with |
| InitialState | `TData` | Starting state when registered |

### Properties

| Property | Type | Description |
|----------|------|-------------|
| EntityId | `int` | This entity's unique ID (GetInstanceID) |
| State | `TData` | Get/set state in the set (protected) |
| IsRegistered | `bool` | Whether currently registered (protected) |

### Events

| Event | Type | Description |
|-------|------|-------------|
| OnStateChanged | `Action<TData, TData>` | Raised when this entity's state changes |

### Virtual methods

| Method | Description |
|--------|-------------|
| OnEnable | Registers to set, subscribes to changes |
| OnDisable | Unsubscribes and unregisters |
| OnBeforeUnregister | Called before unregistration (override for cleanup) |

---

## Example implementation

### State struct

```csharp
[System.Serializable]
public struct EnemyState
{
    public int Health;
    public int MaxHealth;
    public bool IsAggressive;

    public float HealthPercent => (float)Health / MaxHealth;
}
```

### Set ScriptableObject

```csharp
using Tang3cko.ReactiveSO;
using UnityEngine;

[CreateAssetMenu(
    fileName = "EnemyEntitySet",
    menuName = "Game/Enemy Entity Set"
)]
public class EnemyEntitySetSO : ReactiveEntitySetSO<EnemyState>
{
    // Add custom methods if needed
    public int CountLowHealth(float threshold)
    {
        int count = 0;
        ForEach((id, state) => {
            if (state.HealthPercent < threshold) count++;
        });
        return count;
    }
}
```

### Entity MonoBehaviour

```csharp
using Tang3cko.ReactiveSO;
using UnityEngine;

public class Enemy : ReactiveEntity<EnemyState>
{
    [SerializeField] private EnemyEntitySetSO enemySet;
    [SerializeField] private int maxHealth = 100;

    protected override ReactiveEntitySetSO<EnemyState> Set => enemySet;

    protected override EnemyState InitialState => new EnemyState
    {
        Health = maxHealth,
        MaxHealth = maxHealth,
        IsAggressive = false
    };

    protected override void OnEnable()
    {
        base.OnEnable();
        OnStateChanged += HandleStateChanged;
    }

    protected override void OnDisable()
    {
        OnStateChanged -= HandleStateChanged;
        base.OnDisable();
    }

    public void TakeDamage(int damage)
    {
        var state = State;
        state.Health -= damage;
        State = state;

        if (state.Health <= 0)
        {
            Die();
        }
    }

    private void HandleStateChanged(EnemyState oldState, EnemyState newState)
    {
        if (oldState.Health != newState.Health)
        {
            UpdateHealthBar(newState.HealthPercent);
        }
    }

    protected override void OnBeforeUnregister()
    {
        Debug.Log($"Enemy dying with {State.Health} HP");
    }

    private void Die()
    {
        Destroy(gameObject);
    }
}
```

---

## Performance characteristics

| Operation | Complexity |
|-----------|------------|
| Register | O(1) amortized |
| Unregister | O(1) |
| GetData / SetData | O(1) |
| Contains | O(1) |
| ForEach | O(n) |
| CleanupDestroyed | O(n) |

The Sparse Set data structure provides O(1) access while maintaining cache-friendly contiguous storage for iteration.

---

## Thread safety

ReactiveEntitySetSO is **NOT thread-safe**. All operations must be performed on the Unity main thread. Accessing from Jobs, async contexts, or background threads results in undefined behavior.

---

## References

- [Reactive Entity Sets Guide]({{ '/en/guides/reactive-entity-sets' | relative_url }}) - How to use RES
- [Runtime Sets Reference](runtime-set-types) - For simpler object tracking
- [Event Types Reference](event-types) - Event channels used by RES
