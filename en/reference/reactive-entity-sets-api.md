---
layout: default
title: Reactive Entity Sets API
parent: Reference
nav_order: 4
---

# Reactive Entity Sets API

## Purpose

This reference documents the complete API for ReactiveEntitySetSO and ReactiveEntity. You will find the methods, properties, and events for managing per-entity state.

---

## Overview

Reactive entity sets provide three API styles.

| API style | Class | Use case |
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
    where TData : unmanaged
```

TData must be an unmanaged type (value type without managed references). This enables Job System and Burst compatibility.

---

## Properties

| Property | Type | Description |
|----------|------|-------------|
| Count | `int` | Number of registered entities |
| Data | `NativeSlice<TData>` | Read-only access to all state data (Job System compatible) |
| EntityIds | `NativeSlice<int>` | Read-only access to all entity IDs (Job System compatible) |

### Events

| Property | Type | Description |
|----------|------|-------------|
| OnItemAdded | `IntEventChannelSO` | Raised when entity registered (passes ID) |
| OnItemRemoved | `IntEventChannelSO` | Raised when entity unregistered (passes ID) |
| OnDataChanged | `IntEventChannelSO` | Raised when entity data changes (passes ID) |
| OnSetChanged | `VoidEventChannelSO` | Raised on any change |
| OnTraitAdded | `IntEventChannelSO` | Raised when traits are added (passes ID) |
| OnTraitRemoved | `IntEventChannelSO` | Raised when traits are removed (passes ID) |

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

Manually triggers data changed event.

---

## Traits API (generic)

Traits live directly on `ReactiveEntitySetSO<TData>`. `TTraits` is constrained to `unmanaged, Enum` and is type-locked on first use. Mixing different trait enums in the same set throws `InvalidOperationException`.

Traits are stored as a 64-bit `ulong` mask (up to 64 flags).

### Mutation methods

```csharp
public void AddTraits<TTraits>(int id, TTraits traits)
```

Adds traits using bitwise OR. Fires `OnTraitAdded` and `OnSetChanged`.

```csharp
public void RemoveTraits<TTraits>(int id, TTraits traits)
```

Removes traits using bitwise AND-NOT. Fires `OnTraitRemoved` and `OnSetChanged`.

```csharp
public void SetTraits<TTraits>(int id, TTraits traits)
```

Replaces all traits. Fires the appropriate trait event depending on whether the mask increased or decreased.

```csharp
public void ClearTraits(int id)
```

Sets the trait mask to 0. Fires `OnTraitRemoved` and `OnSetChanged` if the entity had traits.

### Query methods

```csharp
public bool HasTraits<TTraits>(int id, TTraits traits)
```

Returns true if ALL specified flags are set.

```csharp
public bool HasAnyTrait<TTraits>(int id, TTraits traits)
```

Returns true if ANY specified flag is set.

```csharp
public TTraits GetTraits<TTraits>(int id)
```

Gets the current traits. Throws `KeyNotFoundException` if the entity is not registered.

```csharp
public bool TryGetTraits<TTraits>(int id, out TTraits traits)
```

Safe version that returns false if the entity is not registered.

### Iteration methods

```csharp
public void WithTraits<TTraits>(TTraits traits, Action<int, TData> callback)
```

Calls callback for every entity where ALL specified flags are set.

```csharp
public void WithAnyTraits<TTraits>(TTraits traits, Action<int, TData> callback)
```

Calls callback for every entity where ANY specified flag is set.

```csharp
public int CountWithTraits<TTraits>(TTraits traits)
```

Returns the count of entities with ALL specified flags set.

```csharp
public int CountWithAnyTraits<TTraits>(TTraits traits)
```

Returns the count of entities with ANY specified flag set.

### Type locking

`TTraits` is type-locked on first use. The first call that specifies a `TTraits` enum type locks the set to that type. Subsequent calls with a different enum type throw `InvalidOperationException`.

### Example

```csharp
[Flags]
public enum EnemyTraits
{
    None      = 0,
    IsAggro   = 1 << 0,
    IsStunned = 1 << 1,
}

public class EnemyAISystem : MonoBehaviour
{
    [SerializeField] private EnemyEntitySetSO enemySet;

    private void Update()
    {
        // Process only aggro enemies
        enemySet.WithTraits<EnemyTraits>(EnemyTraits.IsAggro, (id, state) =>
        {
            // AI logic for aggro enemies
        });
    }

    public void SetAggro(int entityId, bool aggro)
    {
        if (aggro)
            enemySet.AddTraits<EnemyTraits>(entityId, EnemyTraits.IsAggro);
        else
            enemySet.RemoveTraits<EnemyTraits>(entityId, EnemyTraits.IsAggro);
    }

    public int GetAggroCount()
    {
        return enemySet.CountWithTraits<EnemyTraits>(EnemyTraits.IsAggro);
    }
}
```

### Performance

| Operation | Complexity |
|-----------|------------|
| `AddTraits` / `RemoveTraits` / `SetTraits` / `ClearTraits` | O(1) |
| `HasTraits` / `HasAnyTrait` / `GetTraits` / `TryGetTraits` | O(1) |
| `WithTraits` / `WithAnyTraits` | O(n) |
| `CountWithTraits` / `CountWithAnyTraits` | O(n) |

Memory: 8 bytes per entity (64-bit bitmask, up to 64 flags).

---

## Views API

A view is a live-filtered subset of entities in a set. It recalculates membership automatically when entity data or traits change, and fires `OnEnter` / `OnExit` events as entities cross the predicate boundary.

### ViewTrigger enum

```csharp
public enum ViewTrigger
{
    None       = 0,
    DataOnly   = 1,
    TraitsOnly = 2,
    All        = 3,
}
```

| Value | When the view recalculates membership |
|-------|--------------------------------------|
| `None` | Never. Use for manually-triggered views. |
| `DataOnly` | When any entity's data changes. |
| `TraitsOnly` | When any entity's traits change. |
| `All` | On both data and trait changes. |

### CreateView (data-only)

```csharp
public ReactiveView<TData> CreateView(
    Func<TData, bool> predicate,
    ViewTrigger triggerOn = ViewTrigger.DataOnly
)
```

Creates a view that filters entities by data only.

```csharp
var lowHealthView = enemySet.CreateView(
    state => state.Health < state.MaxHealth * 0.3f,
    ViewTrigger.DataOnly
);
```

### CreateView (trait-aware)

```csharp
public ReactiveView<TData> CreateView(
    Func<TData, ulong, bool> predicate,
    ViewTrigger triggerOn,
    ulong observedTraitMask
)
```

Creates a view whose predicate also receives the entity's trait bitmask. The `observedTraitMask` tells the view which trait flags trigger re-evaluation.

Use `TraitMaskUtility.ToUInt64<TTraits>(flags)` to build the mask.

```csharp
ulong aggroMask = TraitMaskUtility.ToUInt64<EnemyTraits>(EnemyTraits.IsAggro);

var aggroLowHealthView = enemySet.CreateView(
    (state, traits) => (traits & aggroMask) != 0 && state.HealthPercent < 0.5f,
    ViewTrigger.All,
    aggroMask
);
```

### ReactiveView\<TData\> members

| Member | Description |
|--------|-------------|
| `Count` | Number of entities currently in the view |
| `Contains(int id)` | O(1) membership check |
| `GetEnumerator()` | Enumerate member IDs (foreach compatible) |
| `OnEnter` | Fires when an entity joins the view |
| `OnExit` | Fires when an entity leaves the view |
| `Dispose()` | Unregisters the view from its parent set |

### Example

```csharp
public class LowHealthSystem : MonoBehaviour
{
    [SerializeField] private EnemyEntitySetSO enemySet;

    private ReactiveView<EnemyState> lowHealthView;

    private void OnEnable()
    {
        lowHealthView = enemySet.CreateView(
            state => state.HealthPercent < 0.3f,
            ViewTrigger.DataOnly
        );

        lowHealthView.OnEnter += OnEnemyEnteredLowHealth;
        lowHealthView.OnExit += OnEnemyLeftLowHealth;
    }

    private void OnDisable()
    {
        lowHealthView.OnEnter -= OnEnemyEnteredLowHealth;
        lowHealthView.OnExit -= OnEnemyLeftLowHealth;
        lowHealthView.Dispose();
    }

    private void OnEnemyEnteredLowHealth(int entityId)
    {
        // Start flee behavior, play warning sound, etc.
    }

    private void OnEnemyLeftLowHealth(int entityId)
    {
        // Cancel flee behavior
    }
}
```

{: .note }
> `OnEnter` and `OnExit` do not fire during `Clear()` or `RestoreSnapshot()`. Membership is rebuilt silently in those cases.

### Performance

| Operation | Complexity |
|-----------|------------|
| `Contains` | O(1) |
| `GetEnumerator` (full iteration) | O(n) |
| `CreateView` | O(n) (evaluates predicate for all current entities) |
| Membership update per entity change | O(v) where v = number of views |

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

For performance-critical code, access the underlying data directly via NativeSlice.

```csharp
var data = entitySet.Data;
for (int i = 0; i < data.Length; i++)
{
    ProcessState(data[i]);
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

- [Reactive entity sets guide]({{ '/en/guides/reactive-entity-sets' | relative_url }}) - How to use reactive entity sets
- [Traits guide]({{ '/en/guides/reactive-entity-sets/traits' | relative_url }}) - How to define and use trait enums
- [Views guide]({{ '/en/guides/reactive-entity-sets/views' | relative_url }}) - How to create and manage views
- [Runtime sets reference](runtime-set-types) - For simpler object tracking
- [Event types reference](event-types) - Event channels used by reactive entity sets
