---
layout: default
title: Traits
parent: Reactive Entity Sets
grand_parent: Guides
nav_order: 3
---

# Traits

---

## Purpose

Traits are per-entity bitmask flags that let you classify and filter entities without storing extra booleans in your state struct.

---

## What are traits?

Each entity in a set can hold up to 64 binary flags, stored as a single 64-bit integer. You define which flags exist by writing a `[Flags]` enum. Common uses are classifying entities for bulk processing — "give me all aggro enemies" or "skip dead ones" — rather than checking fields on each entity in a loop.

Traits work best for classification and filtering. If you need a value that changes frequently (health, position, speed), put it in your state struct where it can be read and written directly. Use traits when the answer is yes-or-no: is this enemy aggro? is this unit stunned?

---

## Step 1: Define a trait enum

The enum must carry `[Flags]` and every non-zero value must be a distinct power of two. Always include a `None = 0` member.

```csharp
using System;

[Flags]
public enum EnemyTraits
{
    None      = 0,
    IsAggro   = 1 << 0,  // 1
    IsDead    = 1 << 1,  // 2
    IsStunned = 1 << 2,  // 4
    IsBoss    = 1 << 3,  // 8
}
```

You can combine flags at the declaration level for convenience.

```csharp
[Flags]
public enum EnemyTraits
{
    None            = 0,
    IsAggro         = 1 << 0,
    IsDead          = 1 << 1,
    IsStunned       = 1 << 2,
    IsBoss          = 1 << 3,
    IsAggroAndBoss  = IsAggro | IsBoss,
}
```

---

## Step 2: Modify traits

Four methods change an entity's trait mask.

```csharp
public class Enemy : ReactiveEntity<EnemyState>
{
    [SerializeField] private EnemyEntitySetSO entitySet;

    public void BecomeAggro()
    {
        // Adds the flag — existing traits are preserved
        entitySet.AddTraits(EntityId, EnemyTraits.IsAggro);
    }

    public void Stun()
    {
        entitySet.AddTraits(EntityId, EnemyTraits.IsStunned);
    }

    public void Unstun()
    {
        // Removes only the specified flag
        entitySet.RemoveTraits(EntityId, EnemyTraits.IsStunned);
    }

    public void Die()
    {
        // Replace the entire mask with exactly these flags
        entitySet.SetTraits(EntityId, EnemyTraits.IsDead);
    }

    public void Respawn()
    {
        // Set mask to 0 — clears all flags at once
        entitySet.ClearTraits(EntityId);
    }
}
```

`AddTraits` performs a bitwise OR, so calling it twice with the same flag is safe. `RemoveTraits` uses bitwise AND-NOT. `SetTraits` replaces the whole mask, discarding any flags not included in the argument. `ClearTraits` sets the mask to zero regardless of current state.

---

## Step 3: Query traits

Two boolean methods let you check flags for a single entity.

```csharp
public class EnemyAI : MonoBehaviour
{
    [SerializeField] private EnemyEntitySetSO entitySet;

    public bool CanAttack(int enemyId)
    {
        // HasTraits returns true only when ALL specified flags are set
        return entitySet.HasTraits(enemyId, EnemyTraits.IsAggro) &&
               !entitySet.HasTraits(enemyId, EnemyTraits.IsStunned);
    }

    public bool IsDisabled(int enemyId)
    {
        // HasAnyTrait returns true when at least ONE specified flag is set
        return entitySet.HasAnyTrait(enemyId, EnemyTraits.IsDead | EnemyTraits.IsStunned);
    }

    public void LogTraits(int enemyId)
    {
        // GetTraits throws if the entity is not registered
        var traits = entitySet.GetTraits<EnemyTraits>(enemyId);
        Debug.Log($"Enemy {enemyId} traits: {traits}");

        // TryGetTraits is safer when the entity may not be present
        if (entitySet.TryGetTraits(enemyId, out EnemyTraits current))
        {
            Debug.Log($"Enemy {enemyId} traits: {current}");
        }
    }
}
```

Use `HasTraits` when you need all flags present — for example, checking that an enemy is both aggro and a boss. Use `HasAnyTrait` when one flag is enough — for example, stopping a system if the entity is dead or stunned.

---

## Step 4: Iterate with trait filters

Four methods let you iterate or count entities based on their traits.

```csharp
public class CombatSystem : MonoBehaviour
{
    [SerializeField] private EnemyEntitySetSO entitySet;

    public void TickAggroEnemies()
    {
        // Visits only entities where ALL specified flags are set
        entitySet.WithTraits(EnemyTraits.IsAggro, (id, state) =>
        {
            ChasePlayer(id, state);
        });
    }

    public void SlowDisabledEnemies()
    {
        // Visits entities where ANY of the specified flags is set
        entitySet.WithAnyTraits(EnemyTraits.IsDead | EnemyTraits.IsStunned, (id, state) =>
        {
            ApplySlowEffect(id);
        });
    }

    public void ShowBossWarning()
    {
        // Count without allocating a collection
        int bossCount = entitySet.CountWithTraits(EnemyTraits.IsBoss);
        if (bossCount > 0)
        {
            ShowBossUI();
        }
    }

    public int GetThreatLevel()
    {
        // Counts entities matching any of the given flags
        return entitySet.CountWithAnyTraits(EnemyTraits.IsAggro | EnemyTraits.IsBoss);
    }
}
```

Both `WithTraits` and `WithAnyTraits` iterate the full set internally, so their cost scales with the number of registered entities. If you only need a count, prefer `CountWithTraits` or `CountWithAnyTraits` over iterating and incrementing manually.

> {: .note }
> Do not call `AddTraits`, `RemoveTraits`, `SetTraits`, or `ClearTraits` inside a `WithTraits` or `WithAnyTraits` callback. Modifying traits during iteration is not supported.

---

## Trait events

`OnTraitAdded` fires after `AddTraits` or `SetTraits` add at least one new flag. `OnTraitRemoved` fires after `RemoveTraits`, `SetTraits`, or `ClearTraits` clear at least one flag. Both pass the entity ID. They are `IntEventChannelSO` assets, assigned in the Inspector the same way as `OnItemAdded`.

```csharp
public class TraitLogger : MonoBehaviour
{
    [SerializeField] private IntEventChannelSO onEnemyTraitAdded;
    [SerializeField] private IntEventChannelSO onEnemyTraitRemoved;
    [SerializeField] private EnemyEntitySetSO entitySet;

    private void OnEnable()
    {
        onEnemyTraitAdded.OnEventRaised += HandleTraitAdded;
        onEnemyTraitRemoved.OnEventRaised += HandleTraitRemoved;
    }

    private void OnDisable()
    {
        onEnemyTraitAdded.OnEventRaised -= HandleTraitAdded;
        onEnemyTraitRemoved.OnEventRaised -= HandleTraitRemoved;
    }

    private void HandleTraitAdded(int entityId)
    {
        if (entitySet.TryGetTraits(entityId, out EnemyTraits traits))
        {
            Debug.Log($"Enemy {entityId} gained traits: {traits}");
        }
    }

    private void HandleTraitRemoved(int entityId)
    {
        Debug.Log($"Enemy {entityId} lost traits");
    }
}
```

Create the event channel assets from:

```text
Create > Reactive SO > Channels > Int Event
```

Assign them to the entity set's `On Trait Added` and `On Trait Removed` fields in the Inspector.

---

## Type locking

The first time you call any trait method on a set (for example, `AddTraits<EnemyTraits>(...)`), the generic type `EnemyTraits` is stored internally. Every subsequent trait call on that set must use the same enum type. Passing a different type throws `InvalidOperationException`.

```csharp
// This locks the set to EnemyTraits
entitySet.AddTraits(1, EnemyTraits.IsAggro);

// Later in another script — fine, same type
entitySet.HasTraits(1, EnemyTraits.IsDead);

// Different enum type — throws InvalidOperationException
entitySet.AddTraits(1, BossTraits.IsEnraged);
```

One set holds one trait enum. If you have enemies with fundamentally different classification needs, use separate entity set assets.

> {: .warning }
> Calling trait methods before any entity is registered still locks the type. The lock happens on the first generic call, not the first registration.

---

## API summary

### Mutation methods

| Method | Description |
|--------|-------------|
| `AddTraits<TTraits>(id, traits)` | Bitwise OR the specified flags into the entity's mask |
| `RemoveTraits<TTraits>(id, traits)` | Bitwise AND-NOT the specified flags from the entity's mask |
| `SetTraits<TTraits>(id, traits)` | Replace the entire trait mask with the given value |
| `ClearTraits(id)` | Set the trait mask to zero |

### Query methods

| Method | Description |
|--------|-------------|
| `HasTraits<TTraits>(id, traits)` | Returns true when ALL specified flags are present |
| `HasAnyTrait<TTraits>(id, traits)` | Returns true when ANY specified flag is present |
| `GetTraits<TTraits>(id)` | Returns the current trait mask; throws if entity not found |
| `TryGetTraits<TTraits>(id, out traits)` | Returns the trait mask without throwing; returns false if not found |

### Iteration methods

| Method | Description |
|--------|-------------|
| `WithTraits<TTraits>(traits, callback)` | Invokes callback for each entity where ALL flags match |
| `WithAnyTraits<TTraits>(traits, callback)` | Invokes callback for each entity where ANY flag matches |
| `CountWithTraits<TTraits>(traits)` | Counts entities where ALL flags match |
| `CountWithAnyTraits<TTraits>(traits)` | Counts entities where ANY flag matches |

### Events

| Event | Type | Fires when |
|-------|------|------------|
| `OnTraitAdded` | `IntEventChannelSO` | Any flag is added to an entity |
| `OnTraitRemoved` | `IntEventChannelSO` | Any flag is removed from an entity |

---

## Performance

| Operation | Complexity |
|-----------|-----------|
| `AddTraits` | O(1) |
| `RemoveTraits` | O(1) |
| `SetTraits` | O(1) |
| `ClearTraits` | O(1) |
| `HasTraits` | O(1) |
| `HasAnyTrait` | O(1) |
| `GetTraits` | O(1) |
| `TryGetTraits` | O(1) |
| `WithTraits` | O(n) |
| `WithAnyTraits` | O(n) |
| `CountWithTraits` | O(n) |
| `CountWithAnyTraits` | O(n) |

Trait data is 8 bytes per entity (one 64-bit bitmask). It is allocated lazily on the first trait call, so sets that never use traits carry no memory overhead.

---

## Next steps

- [Views](views) - Filter entities into reactive subsets using traits and data predicates
- [Patterns](patterns) - See common usage patterns combining traits with events and systems
