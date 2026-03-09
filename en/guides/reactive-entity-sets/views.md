---
layout: default
title: Views
parent: Reactive Entity Sets
grand_parent: Guides
nav_order: 4
---

# Views

---

## Purpose

A `ReactiveView<TData>` is a filtered subset of an entity set that updates itself automatically whenever entity data or traits change.

---

## What are views?

A view holds the IDs of entities that satisfy a predicate. When an entity's data or traits change, the view re-evaluates the predicate and adds or removes the entity from its membership. You subscribe to `OnEnter` and `OnExit` to react to those changes.

Without views, filtering looks like this:

```csharp
// Pull model: you scan the full set every frame
entitySet.ForEach((id, state) =>
{
    if (state.HealthPercent < 0.3f)
        HighlightEnemy(id);
});
```

With a view, the set tells you when membership changes:

```csharp
// Push model: the view notifies you
lowHealthView.OnEnter += id => HighlightEnemy(id);
lowHealthView.OnExit  += id => ClearHighlight(id);
```

The view is evaluated once per entity change rather than once per frame per entity.

---

## Creating a data-only view

Pass a `Func<TData, bool>` predicate to `CreateView`. The default trigger is `ViewTrigger.DataOnly`, which re-evaluates the predicate whenever an entity's data changes.

```csharp
public class EnemyAlertSystem : MonoBehaviour
{
    [SerializeField] private EnemyEntitySetSO entitySet;

    private ReactiveView<EnemyState> lowHealthView;

    private void OnEnable()
    {
        // Include enemies with less than 30% health
        lowHealthView = entitySet.CreateView(
            state => state.HealthPercent < 0.3f,
            ViewTrigger.DataOnly
        );

        lowHealthView.OnEnter += OnEnemyLowHealth;
        lowHealthView.OnExit  += OnEnemyHealthRecovered;
    }

    private void OnDisable()
    {
        lowHealthView?.Dispose();
        lowHealthView = null;
    }

    private void OnEnemyLowHealth(int enemyId)
    {
        Debug.Log($"Enemy {enemyId} is low health");
    }

    private void OnEnemyHealthRecovered(int enemyId)
    {
        Debug.Log($"Enemy {enemyId} recovered");
    }
}
```

`CreateView` evaluates the predicate against every entity currently in the set, so the view is populated immediately on creation.

---

## Creating a trait-aware view

Pass a `Func<TData, ulong, bool>` predicate to filter by both data and traits. The second argument is the entity's raw trait bitmask. Use `TraitMaskUtility.ToUInt64<TTraits>` to convert your enum to the `ulong` you compare against.

```csharp
public class AggroLowHealthSystem : MonoBehaviour
{
    [SerializeField] private EnemyEntitySetSO entitySet;

    private ReactiveView<EnemyState> dangerousEnemyView;

    private void OnEnable()
    {
        ulong aggroMask = TraitMaskUtility.ToUInt64(EnemyTraits.IsAggro);

        // Match enemies that are aggro AND below 50% health
        dangerousEnemyView = entitySet.CreateView(
            predicate: (state, traitMask) =>
                (traitMask & aggroMask) != 0 && state.HealthPercent < 0.5f,
            triggerOn: ViewTrigger.All,
            observedTraitMask: aggroMask
        );

        dangerousEnemyView.OnEnter += id => Debug.Log($"Enemy {id} is dangerous");
        dangerousEnemyView.OnExit  += id => Debug.Log($"Enemy {id} is no longer dangerous");
    }

    private void OnDisable()
    {
        dangerousEnemyView?.Dispose();
        dangerousEnemyView = null;
    }
}
```

The `observedTraitMask` parameter tells the view which trait bits to watch. When only unrelated traits change, the predicate is not re-evaluated. This avoids unnecessary work when a set has many trait types or high-frequency trait changes.

---

## `ViewTrigger` modes

| Mode | Value | When the predicate is re-evaluated |
|------|-------|------------------------------------|
| `None` | 0 | Never after creation — membership is static |
| `DataOnly` | 1 | When an entity's data changes |
| `TraitsOnly` | 2 | When an entity's traits change |
| `All` | 3 | When either data or traits change |

`None` is the right choice for views that never need to update after initialization — for example, a view that groups entities by a fixed property set at registration. Most gameplay filters (health thresholds, position ranges) work well with `DataOnly`. If the predicate depends entirely on trait flags, pick `TraitsOnly`. When both data and traits contribute to membership, `All` covers both triggers.

---

## Membership events

`OnEnter` fires when an entity passes the predicate. `OnExit` fires when an entity fails it. Both pass the entity ID.

```csharp
public class LowHealthUI : MonoBehaviour
{
    [SerializeField] private EnemyEntitySetSO entitySet;
    [SerializeField] private GameObject warningIconPrefab;

    private ReactiveView<EnemyState> lowHealthView;
    private Dictionary<int, GameObject> warningIcons = new();

    private void OnEnable()
    {
        lowHealthView = entitySet.CreateView(state => state.HealthPercent < 0.25f);
        lowHealthView.OnEnter += SpawnWarning;
        lowHealthView.OnExit  += RemoveWarning;
    }

    private void OnDisable()
    {
        lowHealthView?.Dispose();
        lowHealthView = null;
    }

    private void SpawnWarning(int enemyId)
    {
        var icon = Instantiate(warningIconPrefab);
        warningIcons[enemyId] = icon;
    }

    private void RemoveWarning(int enemyId)
    {
        if (warningIcons.TryGetValue(enemyId, out var icon))
        {
            Destroy(icon);
            warningIcons.Remove(enemyId);
        }
    }
}
```

`OnEnter` also fires when an entity registers directly into the view's predicate range, and `OnExit` fires when an entity unregisters while in the view.

---

## Iterating view members

Use `foreach` to iterate all member IDs, or `Contains` for a one-off membership check.

```csharp
public class ThreatHUD : MonoBehaviour
{
    [SerializeField] private EnemyEntitySetSO entitySet;

    private ReactiveView<EnemyState> aggroView;

    private void OnEnable()
    {
        ulong aggroMask = TraitMaskUtility.ToUInt64(EnemyTraits.IsAggro);
        aggroView = entitySet.CreateView(
            (state, mask) => (mask & aggroMask) != 0,
            ViewTrigger.TraitsOnly,
            aggroMask
        );
    }

    private void OnDisable()
    {
        aggroView?.Dispose();
        aggroView = null;
    }

    public void DrawThreatList()
    {
        foreach (int id in aggroView)
        {
            DrawThreatMarker(id);
        }
    }

    public bool IsTargetThreatening(int enemyId)
    {
        // O(1) check — no iteration needed
        return aggroView.Contains(enemyId);
    }
}
```

`Count` gives the current number of members without iterating.

```csharp
int threats = aggroView.Count;
```

---

## Lifecycle

Call `Dispose()` when the view is no longer needed. The standard place is `OnDisable`, which mirrors where you call `Dispose` on NativeContainers.

```csharp
private void OnDisable()
{
    lowHealthView?.Dispose();
    lowHealthView = null;
}
```

`Dispose` removes the view from the parent set's internal list. If you skip it, the set holds a reference to the view indefinitely and continues evaluating its predicate on every entity change. In the Editor, this produces a memory leak warning when the set asset is destroyed at domain reload.

> {: .warning }
> Never call methods on a view after `Dispose`. `Contains` will return false, `Count` will return 0, and iterating will throw an exception because the underlying `NativeHashSet` has been freed.

---

## Bulk operation behavior

`Clear()` and `RestoreSnapshot()` remove or replace all entities at once. For these operations, the view rebuilds its membership silently without raising `OnEnter` or `OnExit`. This is intentional — firing per-entity events for a hundred or more entities during a bulk reset is rarely useful and adds measurable overhead.

> {: .note }
> `OnEnter` and `OnExit` are NOT raised during `Clear()` or `RestoreSnapshot()`. After either call, membership in all views reflects the new state, but no events fire. If you need to react to the bulk change, subscribe to `OnSetChanged` on the entity set instead.

```csharp
// This fires OnSetChanged once, not OnExit for each member
entitySet.Clear();

// This also fires OnSetChanged once after rebuilding
entitySet.RestoreSnapshot(snapshot);
```

---

## API summary

### `ReactiveView<TData>` members

| Member | Description |
|--------|-------------|
| `int Count` | Number of entities currently in the view |
| `bool Contains(int id)` | Returns true if the entity is a current member; O(1) |
| `GetEnumerator()` | Iterates member IDs; compatible with `foreach` |
| `event Action<int> OnEnter` | Fired when an entity enters the view |
| `event Action<int> OnExit` | Fired when an entity exits the view |
| `void Dispose()` | Releases native memory and unregisters from the parent set |

### `CreateView` overloads

| Signature | Default trigger | Use for |
|-----------|-----------------|---------|
| `CreateView(Func<TData, bool> predicate, ViewTrigger triggerOn)` | `DataOnly` | Predicates that depend only on entity data |
| `CreateView(Func<TData, ulong, bool> predicate, ViewTrigger triggerOn, ulong observedTraitMask)` | `All` | Predicates that depend on traits, or on both data and traits |

---

## Performance

| Operation | Complexity |
|-----------|-----------|
| `CreateView` | O(n) — evaluates predicate for all current entities |
| `Contains` | O(1) |
| `GetEnumerator` | O(n) iteration |
| Membership update per entity change | O(v) where v = number of active views on the set |

Each view allocates a `NativeHashSet<int>` backed by persistent native memory. Keep the number of views per set reasonable. Three to five views on a single set is typical; dozens may add measurable per-entity overhead.

---

## Next steps

- [Patterns](patterns) - See how views combine with event channels and systems
- [Job System](job-system) - Use `NativeSlice` data alongside views for Burst-compatible processing
- [Observability](observability) - Monitor view membership counts at runtime
