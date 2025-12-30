---
layout: default
title: Best Practices
parent: Reactive Entity Sets
grand_parent: Guides
nav_order: 4
---

# Best Practices

---

## Purpose

This page covers best practices for using Reactive Entity Sets effectively. You will learn about state struct design, update patterns, memory management, and troubleshooting common issues.

---

## State struct design

### Keep state structs simple

Use primitive types for best performance.

```csharp
// Good: Simple value types
[Serializable]
public struct EntityState
{
    public int Health;
    public float Speed;
    public Vector3 LastPosition;
}
```

```csharp
// Bad: Reference types cause allocation
[Serializable]
public struct EntityState
{
    public List<int> Modifiers;  // Avoid
    public string Name;           // Avoid if possible
}
```

### Computed properties are fine

Read-only properties that derive from stored values are acceptable.

```csharp
[Serializable]
public struct EntityState
{
    public int Health;
    public int MaxHealth;

    // Good: Computed from stored values
    public float HealthPercent => MaxHealth > 0 ? (float)Health / MaxHealth : 0f;
    public bool IsDead => Health <= 0;
}
```

---

## Update patterns

### Use UpdateData for atomic modifications

```csharp
// Good: Atomic update
entitySet.UpdateData(this, state => {
    state.Health -= damage;
    state.IsStunned = damage > 50;
    return state;
});
```

```csharp
// Bad: Non-atomic, state may change between get and set
var state = entitySet.GetData(this);
state.Health -= damage;
entitySet.SetData(this, state);
```

### Don't mutate state directly

Direct struct mutation does not update the set.

```csharp
// Bad: Direct mutation doesn't trigger events
var state = entitySet.GetData(entityId);
state.Health -= 10;  // Set is NOT updated!
```

```csharp
// Good: Use SetData or UpdateData
entitySet.UpdateData(entityId, state => {
    state.Health -= 10;
    return state;
});
```

---

## Subscription management

### Always unsubscribe

```csharp
// Good: Balanced subscription
private void OnEnable()
{
    entitySet.SubscribeToEntity(entityId, OnStateChanged);
}

private void OnDisable()
{
    entitySet.UnsubscribeFromEntity(entityId, OnStateChanged);
}
```

```csharp
// Bad: Memory leak
private void Start()
{
    entitySet.SubscribeToEntity(entityId, OnStateChanged);
}
// Missing unsubscribe in OnDisable
```

### Initialize state on subscribe

Update your UI immediately after subscribing.

```csharp
private void OnEnable()
{
    entitySet.SubscribeToEntity(entityId, OnStateChanged);

    // Initialize immediately
    if (entitySet.TryGetData(entityId, out var state))
    {
        UpdateDisplay(state);
    }
}
```

---

## Scene persistence

### State survives scene changes

Entity data is stored in ScriptableObjects and persists across scene loads.

{: .warning }
> **Important**: ScriptableObjects may be unloaded from memory if not referenced by any active object during a scene transition. To prevent data loss, use `ReactiveEntitySetHolder` in a Manager Scene. See [Persistence](persistence) for details.

```csharp
// Scene A: Enemy registers
entitySet.Register(enemyId, initialState);

// Scene B loads
// State is still accessible
if (entitySet.TryGetData(enemyId, out var state))
{
    // State persists
}
```

### Clear when appropriate

Clean up when the game context changes.

```csharp
public class GameManager : MonoBehaviour
{
    [SerializeField] private EnemyEntitySetSO entitySet;

    public void RestartLevel()
    {
        entitySet.Clear();
        SceneManager.LoadScene(SceneManager.GetActiveScene().name);
    }
}
```

### Play mode vs build behavior

ScriptableObject data persists during a play session but resets when exiting Play Mode (Editor) or restarting the application (builds). For permanent persistence, serialize to PlayerPrefs, JSON files, or a database.

---

## Performance guidelines

### When RES excels

- Less than 10% of entities change per frame
- Entity count is moderate (hundreds to low thousands)
- Multiple systems need to react to the same state changes

### When to consider alternatives

- All entities update every frame
- Entity count is in the tens of thousands
- Raw performance is the top priority

### Avoid high-frequency data

```csharp
// Reconsider if this changes every frame
[Serializable]
public struct EntityState
{
    public float AnimationTime;  // Changes every frame - avoid
    public Vector3 Velocity;     // Changes every frame - avoid
}
```

See [Data Guidelines]({{ '/en/design-philosophy/reactive-entity-sets/data-guidelines' | relative_url }}) for detailed guidance.

---

## Troubleshooting

### State not updating

Ensure you use `SetData` or `UpdateData`. Direct struct mutation does not update the set.

```csharp
// Wrong
var state = entitySet.GetData(entityId);
state.Health = 50;  // Set not updated!

// Correct
entitySet.SetData(entityId, new EnemyState { Health = 50 });
```

### Events not firing

1. Verify the entity is registered with `Contains()`
2. Check that you subscribed before the state change
3. Ensure event channels are assigned in the Inspector

### Entity not found

1. Check if `OnEnable` runs before you query the entity
2. Verify the entity ID is correct (`GetInstanceID()` changes each play session)
3. Use `TryGetData` instead of `GetData` for safe access

```csharp
// Safe access pattern
if (entitySet.TryGetData(entityId, out var state))
{
    // Use state
}
else
{
    // Handle missing entity
}
```

### Memory leaks

Always unsubscribe in `OnDisable`.

```csharp
private void OnDisable()
{
    entitySet.UnsubscribeFromEntity(entityId, OnStateChanged);
}
```

### Stale subscriptions after scene load

When a scene unloads, GameObjects are destroyed but ScriptableObject data persists. If you don't unsubscribe in OnDisable, callbacks may point to destroyed objects.

```csharp
// Always unsubscribe in OnDisable
private void OnDisable()
{
    if (trackedEntityId != 0)
    {
        entitySet.UnsubscribeFromEntity(trackedEntityId, OnStateChanged);
    }
}
```

---

## Summary

| Practice | Description |
|----------|-------------|
| Simple structs | Use primitive types, avoid reference types |
| Atomic updates | Use UpdateData for multiple field changes |
| Balanced subscriptions | Always unsubscribe in OnDisable |
| Initialize on subscribe | Update display immediately after subscribing |
| Safe access | Use TryGetData for nullable access |
| Scene awareness | Clear sets when game context changes |

---

## Related documentation

- [RES Design Philosophy]({{ '/en/design-philosophy/reactive-entity-sets/' | relative_url }}) - Conceptual foundations
- [Data Guidelines]({{ '/en/design-philosophy/reactive-entity-sets/data-guidelines' | relative_url }}) - What data belongs in RES
