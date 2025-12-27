---
layout: default
title: Basic Usage
parent: Reactive Entity Sets
grand_parent: Guides
nav_order: 1
---

# Basic Usage

---

## Purpose

This page walks through the steps to set up and use Reactive Entity Sets in your project. You will create a state struct, entity set asset, and entity component.

---

## Step 1: Define your state struct

Create a struct to hold entity data. It must be `[System.Serializable]`.

```csharp
using System;

[Serializable]
public struct EnemyState
{
    public int Health;
    public int MaxHealth;
    public bool IsStunned;

    public float HealthPercent => MaxHealth > 0 ? (float)Health / MaxHealth : 0f;
    public bool IsDead => Health <= 0;
}
```

---

## Step 2: Create a reactive entity set asset

Create a class inheriting from `ReactiveEntitySetSO<T>`.

```csharp
using Tang3cko.ReactiveSO;
using UnityEngine;

[CreateAssetMenu(
    fileName = "EnemyEntitySet",
    menuName = "Reactive SO/Entity Sets/Enemy"
)]
public class EnemyEntitySetSO : ReactiveEntitySetSO<EnemyState>
{
    // Base class provides all functionality
}
```

Then create an asset in the Project window. Select the following menu path.

```text
Create > Reactive SO > Entity Sets > Enemy
```

---

## Step 3: Create event channels (optional)

If you need notifications for set-level changes, create event channels.

```text
Create > Reactive SO > Channels > Int Event
```

Assign them to the entity set's fields. The available event fields are described below.

- **On Item Added** - Fires when entity registers
- **On Item Removed** - Fires when entity unregisters
- **On Data Changed** - Fires when any entity's data changes
- **On Set Changed** - Fires on any change

---

## Step 4: Create your entity component

Use the `ReactiveEntity<T>` base class for automatic lifecycle management.

```csharp
using Tang3cko.ReactiveSO;
using UnityEngine;

public class Enemy : ReactiveEntity<EnemyState>
{
    [SerializeField] private EnemyEntitySetSO entitySet;
    [SerializeField] private int maxHealth = 100;

    // Required: specify which set to use
    protected override ReactiveEntitySetSO<EnemyState> Set => entitySet;

    // Required: specify initial state
    protected override EnemyState InitialState => new EnemyState
    {
        Health = maxHealth,
        MaxHealth = maxHealth,
        IsStunned = false
    };

    public void TakeDamage(int damage)
    {
        var state = State;
        state.Health = Mathf.Max(0, state.Health - damage);
        State = state;  // Automatically triggers events

        if (state.IsDead)
        {
            Destroy(gameObject);  // OnDisable unregisters automatically
        }
    }
}
```

---

## Step 5: Query entity data

Access entity data from any script.

```csharp
public class EnemyManager : MonoBehaviour
{
    [SerializeField] private EnemyEntitySetSO entitySet;

    public int GetTotalEnemyHealth()
    {
        int total = 0;
        entitySet.ForEach((id, state) => {
            total += state.Health;
        });
        return total;
    }
}
```

---

## API reference

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `Count` | `int` | Number of registered entities |
| `EntityIds` | `ArraySegment<int>` | All registered entity IDs |
| `Data` | `ArraySegment<TData>` | All entity data |

### Methods

| Method | Description |
|--------|-------------|
| `Register(owner, data)` | Register entity with initial state |
| `Unregister(owner)` | Remove entity from set |
| `GetData(owner)` | Get entity's current state |
| `TryGetData(owner, out data)` | Safely get entity's state |
| `SetData(owner, data)` | Update entity's state |
| `UpdateData(owner, func)` | Update state with function |
| `GetDataRef(id)` | Get reference to state for direct modification |
| `NotifyDataChanged(owner)` | Manually notify after GetDataRef modification |
| `Contains(owner)` | Check if entity exists |
| `Clear()` | Remove all entities |
| `ForEach(action)` | Iterate all entities |
| `SubscribeToEntity(id, callback)` | Subscribe to entity changes |
| `UnsubscribeFromEntity(id, callback)` | Unsubscribe from entity |

---

## Performance characteristics

Reactive Entity Sets use a Sparse Set data structure.

| Operation | Time Complexity |
|-----------|-----------------|
| Register | O(1) |
| Unregister | O(1) |
| GetData | O(1) |
| SetData | O(1) |
| Iteration | O(n) |

The Sparse Set allocates memory in pages, only creating pages for used ID ranges. This is efficient for sparse ID distributions.

---

## Next steps

- [Events](events) - Learn about per-entity and set-level subscriptions
- [Patterns](patterns) - See common usage patterns
