---
layout: default
title: Runtime set types
parent: Reference
nav_order: 3
---

# Runtime set types

## Purpose

This reference documents all built-in runtime set types. You will find the API reference, use cases, and code examples for managing dynamic object collections.

---

## Type overview

| Type | Element Type | Special Features |
|------|--------------|------------------|
| GameObjectRuntimeSetSO | `GameObject` | `DestroyItems()` method |
| TransformRuntimeSetSO | `Transform` | - |

---

## Common API

All runtime set types share this API.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| Items | `IReadOnlyList<T>` | Read-only access to all items |
| Count | `int` | Number of items in the set |

### Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `Add(T item)` | `void` | Add item (ignores duplicates) |
| `Remove(T item)` | `bool` | Remove item, returns success |
| `Clear()` | `void` | Remove all items |
| `Contains(T item)` | `bool` | Check if item is in set |

### Events

| Event | Type | Description |
|-------|------|-------------|
| onItemsChanged | `VoidEventChannelSO` | Raised when items added/removed |
| onCountChanged | `IntEventChannelSO` | Raised with new count |

### Editor Events

| Event | Description |
|-------|-------------|
| ItemsChanged | Action callback for editor refresh |
| OnAnyOperationPerformed | Static event for monitoring |

---

## GameObjectRuntimeSetSO

Tracks GameObject references. Most commonly used runtime set type.

### Create

```text
Create > Reactive SO > Runtime Sets > GameObject Runtime Set
```

### Basic usage

```csharp
[SerializeField] private GameObjectRuntimeSetSO activeEnemies;

// Registration (in enemy script)
private void OnEnable()
{
    activeEnemies?.Add(gameObject);
}

private void OnDisable()
{
    activeEnemies?.Remove(gameObject);
}
```

### Iteration

```csharp
// Access all items
foreach (var enemy in activeEnemies.Items)
{
    if (enemy != null)
    {
        // Process enemy
    }
}

// Check count
if (activeEnemies.Count == 0)
{
    OnAllEnemiesDefeated();
}
```

### DestroyItems method

GameObjectRuntimeSetSO includes a special method to destroy all tracked objects.

```csharp
// Destroy all items and clear the set
activeEnemies.DestroyItems();
```

This is useful for wave-based games or cleanup scenarios.

### Common use cases

- Active enemies in scene
- Collectible pickups
- Interactable objects
- Projectiles in flight
- Spawned items

---

## TransformRuntimeSetSO

Tracks Transform references. Use when you need transform data without full GameObject overhead.

### Create

```text
Create > Reactive SO > Runtime Sets > Transform Runtime Set
```

### Basic usage

```csharp
[SerializeField] private TransformRuntimeSetSO waypoints;

// Registration
private void OnEnable()
{
    waypoints?.Add(transform);
}

private void OnDisable()
{
    waypoints?.Remove(transform);
}
```

### Iteration

```csharp
// Find nearest waypoint
Transform nearest = null;
float nearestDist = float.MaxValue;

foreach (var waypoint in waypoints.Items)
{
    if (waypoint == null) continue;

    float dist = Vector3.Distance(transform.position, waypoint.position);
    if (dist < nearestDist)
    {
        nearestDist = dist;
        nearest = waypoint;
    }
}
```

### Common use cases

- Waypoints for pathfinding
- Spawn points
- Patrol route nodes
- Camera targets
- Reference points

---

## Subscribing to changes

Use event channels to react to collection changes.

### Setup

1. Create a VoidEventChannelSO for `onItemsChanged`
2. Create an IntEventChannelSO for `onCountChanged`
3. Assign them to the runtime set in the Inspector

### Code

```csharp
[SerializeField] private IntEventChannelSO onEnemyCountChanged;

private void OnEnable()
{
    onEnemyCountChanged.OnEventRaised += HandleCountChange;
}

private void OnDisable()
{
    onEnemyCountChanged.OnEventRaised -= HandleCountChange;
}

private void HandleCountChange(int count)
{
    enemyCountText.text = $"Enemies: {count}";
}
```

---

## Duplicate handling

Runtime sets prevent duplicates automatically. Calling `Add()` with an existing item does nothing.

```csharp
enemies.Add(enemy);  // Added (count: 1)
enemies.Add(enemy);  // Ignored (count: 1)
enemies.Add(enemy);  // Ignored (count: 1)
```

---

## Null handling

Runtime sets do not accept null items. Calling `Add(null)` is ignored.

```csharp
enemies.Add(null);  // Ignored, no error
```

Items that become null (destroyed objects) remain in the set until explicitly removed or cleaned up.

---

## Creating custom types

Create a custom runtime set type by extending `RuntimeSetSO<T>`.

```csharp
using Tang3cko.ReactiveSO;
using UnityEngine;

[CreateAssetMenu(
    fileName = "EnemyRuntimeSet",
    menuName = "Reactive SO/Runtime Sets/Enemy Runtime Set"
)]
public class EnemyRuntimeSetSO : RuntimeSetSO<Enemy>
{
    // Inherits all base functionality

    // Add custom methods if needed
    public Enemy GetRandomEnemy()
    {
        if (Count == 0) return null;
        return Items[Random.Range(0, Count)];
    }
}
```

### Notes for custom types

- Use reference types that derive from UnityEngine.Object for proper null checking
- Consider adding utility methods specific to your type
- Custom types work with Runtime Set Monitor

---

## Best practices

### Registration pattern

Always register in `OnEnable` and unregister in `OnDisable`.

```csharp
// ✅ Good: OnEnable/OnDisable pair
private void OnEnable() => enemies?.Add(gameObject);
private void OnDisable() => enemies?.Remove(gameObject);

// ❌ Bad: Start/OnDestroy pair
private void Start() => enemies?.Add(gameObject);
private void OnDestroy() => enemies?.Remove(gameObject);
```

Using `OnDestroy` can leave orphaned entries when objects are disabled but not destroyed.

### Null-conditional operator

Use `?.` when the set might not be assigned.

```csharp
// ✅ Good: Safe if not assigned
activeEnemies?.Add(gameObject);

// ❌ Bad: NullReferenceException if not assigned
activeEnemies.Add(gameObject);
```

### Null checks during iteration

Always check for null when iterating, as objects may be destroyed.

```csharp
foreach (var item in runtimeSet.Items)
{
    if (item == null) continue;
    // Safe to use item
}
```

---

## References

- [Runtime Sets Guide]({{ '/en/guides/runtime-sets' | relative_url }}) - How to use runtime sets
- [Runtime Set Monitor]({{ '/en/debugging/monitor' | relative_url }}) - Debug set contents
- [Debugging Overview]({{ '/en/debugging/' | relative_url }}) - All debugging tools
