---
layout: default
title: Runtime sets
parent: Guides
nav_order: 3
---

# Runtime sets

{: .note }
> Runtime Sets are available since v1.1.0.

---

## Purpose

This guide explains how to use Runtime Sets to track collections of objects without singleton managers. You will learn the registration pattern, how to query active objects, and when to choose Runtime Sets over alternatives.

---

## What are runtime sets?

Runtime Sets are ScriptableObject-based collections that track objects created and destroyed at runtime. Objects register themselves when enabled and unregister when disabled.

```csharp
// Objects register themselves
private void OnEnable() => activeEnemies?.Add(gameObject);
private void OnDisable() => activeEnemies?.Remove(gameObject);

// Query active objects anywhere
int count = activeEnemies.Count;
foreach (var enemy in activeEnemies.Items) { ... }
```

This eliminates the need for singleton managers like `EnemyManager.Instance`.

---

## When to use runtime sets

### Use runtime sets when

- You need to track **dynamically spawned objects** (enemies, pickups, projectiles)
- You want to **avoid singleton managers**
- Multiple systems need to **query the same collection**
- You need to check "all active X" **without `FindObjectsOfType`**

### Use C# List<T> when

- The collection **belongs to one GameObject** only
- You access it **every frame** (performance critical)
- You don't need **cross-system visibility**
- You don't need **event notifications**

### Quick decision guide

| Scenario | Use |
|----------|-----|
| Track all enemies in level | Runtime Set |
| Enemy's personal waypoint list | C# List<T> |
| Track all pickups for minimap | Runtime Set |
| Inventory items in player | C# List<T> |

---

## Available types

| Type | Tracks | Use Case |
|------|--------|----------|
| GameObject Runtime Set | `GameObject` | Most common, spawned entities |
| Transform Runtime Set | `Transform` | Position-based queries, waypoints |

---

## Basic usage

### Step 1: Create a runtime set asset

Right-click in the Project window and select the following menu path.

```text
Create > Reactive SO > Runtime Sets > GameObject Runtime Set
```

Name it descriptively, such as `ActiveEnemies` or `SpawnedPickups`.

### Step 2: Create event channels (optional)

If you need notifications when the collection changes, create an event channel.

```text
Create > Reactive SO > Channels > Void Event
```

Assign it to the runtime set's **On Items Changed** field.

### Step 3: Objects register themselves

Use the OnEnable/OnDisable pattern:

```csharp
using Tang3cko.ReactiveSO;
using UnityEngine;

public class Enemy : MonoBehaviour
{
    [SerializeField] private GameObjectRuntimeSetSO activeEnemies;

    private void OnEnable()
    {
        activeEnemies?.Add(gameObject);
    }

    private void OnDisable()
    {
        activeEnemies?.Remove(gameObject);
    }
}
```

### Step 4: Query the collection

Access active objects from anywhere:

```csharp
public class WaveManager : MonoBehaviour
{
    [SerializeField] private GameObjectRuntimeSetSO activeEnemies;
    [SerializeField] private VoidEventChannelSO onWaveComplete;

    private void Update()
    {
        if (activeEnemies.Count == 0)
        {
            onWaveComplete?.RaiseEvent();
        }
    }
}
```

---

## API reference

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `Count` | `int` | Number of items in the set |
| `Items` | `IReadOnlyList<T>` | Read-only access to all items |

### Methods

| Method | Description |
|--------|-------------|
| `Add(T item)` | Add an item (prevents duplicates) |
| `Remove(T item)` | Remove an item |
| `Contains(T item)` | Check if item exists |
| `Clear()` | Remove all items |
| `DestroyItems()` | Destroy all GameObjects and clear (GameObject sets only) |

---

## Common patterns

### Pattern 1: Wave-based spawning

Track enemies to detect when wave is complete:

```csharp
public class WaveSpawner : MonoBehaviour
{
    [SerializeField] private GameObject enemyPrefab;
    [SerializeField] private GameObjectRuntimeSetSO activeEnemies;
    [SerializeField] private VoidEventChannelSO onWaveComplete;

    public void SpawnWave(int count)
    {
        for (int i = 0; i < count; i++)
        {
            // Enemies register themselves in OnEnable
            Instantiate(enemyPrefab, GetSpawnPosition(), Quaternion.identity);
        }
    }

    private void Update()
    {
        if (activeEnemies.Count == 0)
        {
            onWaveComplete?.RaiseEvent();
        }
    }
}
```

### Pattern 2: Find nearest object

Query without `FindObjectsOfType`:

```csharp
public GameObject GetNearestEnemy(Vector3 position)
{
    GameObject nearest = null;
    float minDistance = float.MaxValue;

    foreach (var enemy in activeEnemies.Items)
    {
        if (enemy == null) continue;

        float distance = Vector3.Distance(position, enemy.transform.position);
        if (distance < minDistance)
        {
            minDistance = distance;
            nearest = enemy;
        }
    }

    return nearest;
}
```

### Pattern 3: Pickup collection

Track collectible items:

```csharp
public class Pickup : MonoBehaviour
{
    [SerializeField] private GameObjectRuntimeSetSO activePickups;

    private void OnEnable()
    {
        activePickups?.Add(gameObject);
    }

    private void OnDisable()
    {
        activePickups?.Remove(gameObject);
    }

    public void Collect()
    {
        // Award points, play sound, etc.
        Destroy(gameObject);  // OnDisable removes from set
    }
}
```

### Pattern 4: Level cleanup

Destroy all spawned objects at once:

```csharp
public class LevelManager : MonoBehaviour
{
    [SerializeField] private GameObjectRuntimeSetSO spawnedEnemies;
    [SerializeField] private GameObjectRuntimeSetSO spawnedPickups;

    public void EndLevel()
    {
        // Destroy all tracked objects
        spawnedEnemies.DestroyItems();
        spawnedPickups.DestroyItems();
    }
}
```

---

## Subscribing to changes

React when items are added or removed:

```csharp
public class EnemyCounter : MonoBehaviour
{
    [SerializeField] private GameObjectRuntimeSetSO activeEnemies;
    [SerializeField] private VoidEventChannelSO onEnemiesChanged;
    [SerializeField] private Text countText;

    private void OnEnable()
    {
        onEnemiesChanged.OnEventRaised += UpdateDisplay;
        UpdateDisplay();
    }

    private void OnDisable()
    {
        onEnemiesChanged.OnEventRaised -= UpdateDisplay;
    }

    private void UpdateDisplay()
    {
        countText.text = $"Enemies: {activeEnemies.Count}";
    }
}
```

---

## Inspector features

During Play Mode, select a Runtime Set asset to see:

- **Live item list** - All currently registered objects
- **Click to ping** - Click any item to highlight in Hierarchy
- **Count display** - Current number of items

<!-- TODO: Add screenshot of Runtime Set Inspector during Play Mode showing live item list -->

Items are automatically cleared when exiting Play Mode.

---

## Best practices

### Always use OnEnable/OnDisable

This ensures automatic registration and cleanup:

```csharp
// ✅ Good: Balanced registration
private void OnEnable()
{
    runtimeSet?.Add(gameObject);
}

private void OnDisable()
{
    runtimeSet?.Remove(gameObject);
}
```

```csharp
// ❌ Bad: Only registers, never unregisters
private void Start()
{
    runtimeSet?.Add(gameObject);
}
```

### Null-check when iterating

Items may be destroyed between checks:

```csharp
foreach (var enemy in activeEnemies.Items)
{
    if (enemy == null) continue;  // Important!
    // Use enemy
}
```

### Clear on scene transitions

Call `Clear()` or `DestroyItems()` when changing scenes:

```csharp
public void LoadNextLevel()
{
    activeEnemies.Clear();  // Prevent stale references
    SceneManager.LoadScene("NextLevel");
}
```

### Use descriptive names

```csharp
// ✅ Good: Clear what it tracks
ActiveEnemies
SpawnedPickups
RegisteredWaypoints

// ❌ Bad: Ambiguous
Enemies
Items
Set1
```

---

## Comparison with alternatives

| Feature | Runtime Sets | Singleton Manager | List in MonoBehaviour |
|---------|--------------|-------------------|----------------------|
| Inspector visibility | Yes | No | Limited |
| Event notifications | Automatic | Manual | Manual |
| Testability | High | Low | Medium |
| Cross-scene persistence | Yes | Yes | No |
| Coupling | Low | High | Medium |

---

## Creating custom runtime sets

For custom types, inherit from `RuntimeSetSO<T>`:

```csharp
[CreateAssetMenu(
    fileName = "EnemyRuntimeSet",
    menuName = "Reactive SO/Runtime Sets/Enemy Runtime Set"
)]
public class EnemyRuntimeSetSO : RuntimeSetSO<Enemy>
{
    // Add custom methods if needed
    public Enemy GetStrongestEnemy()
    {
        return Items
            .Where(e => e != null)
            .OrderByDescending(e => e.Health)
            .FirstOrDefault();
    }
}
```

Use in your component:

```csharp
public class Enemy : MonoBehaviour
{
    [SerializeField] private EnemyRuntimeSetSO activeEnemies;
    public int Health { get; private set; } = 100;

    private void OnEnable() => activeEnemies?.Add(this);
    private void OnDisable() => activeEnemies?.Remove(this);
}
```

---

## Troubleshooting

### Items not appearing in Inspector

Runtime Set contents are only visible during Play Mode.

### Objects not registering

Ensure you call `Add()` in `OnEnable` and the runtime set is assigned in the Inspector.

### Null references after scene change

Call `Clear()` before changing scenes to remove stale references.

### Count not updating in UI

Subscribe to the **On Items Changed** event channel rather than polling every frame.

---

## References

- [Event Channels Guide](event-channels) - For notifications
- [Variables Guide](variables) - For shared state
- [Reactive Entity Sets Guide](reactive-entity-sets) - For entity data with IDs
