---
layout: default
title: Data Guidelines
parent: RES Design
grand_parent: Design Philosophy
nav_order: 3
---

# Data Guidelines

---

## Purpose

This page explains what data should go into a ReactiveEntitySet and what should stay in GameObjects. Following these guidelines leads to cleaner architecture and fewer bugs.

---

## The fundamental rule

> **Only include data that is computed or managed OUTSIDE of GameObjects.**

This rule determines whether data belongs in RES or in a MonoBehaviour.

---

## Data ownership examples

| Data | Computed by | Belongs in RES? | Reason |
|------|-------------|-----------------|--------|
| Health | Damage system (external) | Yes | External logic modifies it |
| MaxHealth | Configuration/initialization | Yes | Reference data |
| SpeedMultiplier | Buff system (external) | Yes | External logic modifies it |
| IsStunned | Status effect system | Yes | External logic controls it |
| KilledByPlayer | Death system | Yes | External flag |
| **Position** | Transform (GameObject) | **No** | Owned by GameObject |
| **Rotation** | Transform (GameObject) | **No** | Owned by GameObject |
| **Scale** | Transform (GameObject) | **No** | Owned by GameObject |

---

## Why Position is excluded

Position seems like important entity data, but it usually does not belong in RES.

### Reasons

1. **Ownership**: The GameObject's Transform component owns position
2. **Update frequency**: Position often changes every frame
3. **Dual source of truth**: Storing in RES creates synchronization problems
4. **No benefit**: Transform already provides this data to anyone who needs it

### The exception

Network games where the **server computes position** are different.

```csharp
// Server authoritative movement
// Server calculates position, sends to clients
// RES becomes the authoritative source

[Serializable]
public struct NetworkEntityState
{
    public Vector3 Position;      // Server-computed, OK in RES
    public Quaternion Rotation;   // Server-computed, OK in RES
    public int Health;
}
```

In this case, the GameObject's Transform is a **view** of the authoritative position in RES.

---

## Design checklist

Before adding data to RES, ask these questions.

### 1. Ownership

**Is this data computed by EXTERNAL logic?**

- Yes → Consider RES
- No (GameObject owns it) → Keep as GameObject field

### 2. Change frequency

**Does this change less than 10% of entities per frame?**

- Yes → RES is efficient
- No → Traditional approach may be better

### 3. Observability

**Do multiple systems need to react to changes?**

- Yes → RES provides built-in events
- No → May not need RES

### 4. Scene independence

**Does this data need to persist across scene loads?**

- Yes → RES provides this automatically
- No → Either approach works

---

## Data/Logic separation

RES naturally enforces a clean architectural pattern.

### The pattern

```
State (struct)        : Data only, no methods
Calculation logic     : Pure functions in separate classes
RES                   : Storage and notification
GameObject            : Visualization and reaction
```

### Why this works

State structs have public fields but no business logic.

```csharp
[Serializable]
public struct EnemyState
{
    public int Health;
    public int MaxHealth;
    public bool IsStunned;

    // Properties are OK
    public float HealthPercent => MaxHealth > 0 ? (float)Health / MaxHealth : 0f;

    // But NO methods that modify state
    // void TakeDamage(int damage) ← Don't do this
}
```

Calculation logic lives in separate classes.

```csharp
public static class DamageCalculator
{
    public static EnemyState ApplyDamage(EnemyState state, int damage, bool isCritical)
    {
        int finalDamage = isCritical ? damage * 2 : damage;
        state.Health = Mathf.Max(0, state.Health - finalDamage);
        return state;
    }
}
```

Usage ties them together.

```csharp
enemySet.UpdateData(enemyId, state =>
    DamageCalculator.ApplyDamage(state, damage, isCritical));
```

### Benefits

| Benefit | Description |
|---------|-------------|
| Testability | Pure functions are easy to unit test |
| Reusability | Same logic works with any entity |
| Clarity | Clear separation of concerns |
| Thread safety | Pure functions have no side effects |

---

## Correspondence with other paradigms

This data/logic separation aligns with modern architectural patterns.

| Paradigm | Data | Logic |
|----------|------|-------|
| Functional | Immutable data | Pure functions |
| ECS | Component | System |
| Redux | State | Reducer |
| **RES** | **State struct** | **Calculation classes** |

If you are familiar with any of these patterns, the RES approach will feel natural.

---

## Async processing potential

The data/logic separation enables future async processing.

- **State (struct)**: Copyable, can be passed between threads
- **Calculation logic**: Pure functions, thread-safe
- **RES update**: Single synchronization point on main thread

Heavy computations can run off the main thread, with results applied to RES when complete.

---

## Common mistakes

### Mistake 1: Storing Transform data

```csharp
// Don't do this (usually)
[Serializable]
public struct EntityState
{
    public Vector3 Position;    // Transform owns this
    public Quaternion Rotation; // Transform owns this
    public int Health;
}
```

### Mistake 2: Putting logic in state struct

```csharp
// Don't do this
[Serializable]
public struct EntityState
{
    public int Health;

    public void TakeDamage(int damage)  // Logic doesn't belong here
    {
        Health -= damage;
    }
}
```

### Mistake 3: Including frequently changing data

```csharp
// Reconsider if this changes every frame
[Serializable]
public struct EntityState
{
    public float AnimationTime;  // Changes every frame
    public Vector3 Velocity;     // Changes every frame
}
```

---

## Summary

| Guideline | Description |
|-----------|-------------|
| External ownership | Only RES data that external systems compute |
| No Transform data | Position/Rotation belong to GameObject (usually) |
| Low change frequency | Less than 10% of entities per frame |
| Data/Logic separation | State structs + pure function calculators |
| Exception for networking | Server-authoritative data can include position |

---

## Next steps

- [Set Theory Foundation](set-theory) - Mathematical basis and View theory
- [Reactive Entity Sets Guide]({{ '/en/guides/reactive-entity-sets/' | relative_url }}) - Implementation details
