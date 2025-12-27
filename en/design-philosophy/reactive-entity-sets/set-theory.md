---
layout: default
title: Set Theory Foundation
parent: RES Design
grand_parent: Design Philosophy
nav_order: 4
---

# Set Theory Foundation

---

## Purpose

This page explains the mathematical foundations of Reactive Entity Sets. Understanding these concepts provides deeper insight into why RES works the way it does and prepares you for advanced features like Views.

---

## ReactiveEntitySet as a mathematical set

A ReactiveEntitySet can be understood as a mathematical set S where each element is a pair of (Entity ID, Data).

```
S = { (id₁, data₁), (id₂, data₂), ..., (idₙ, dataₙ) }
```

### Key properties

| Property | Description |
|----------|-------------|
| **Uniqueness** | Each entity ID appears at most once |
| **Membership** | An entity is either in the set or not (no partial membership) |
| **Data association** | Each member has exactly one associated data value |

---

## Operations and closure

### Closure property

A set family S is **closed** under an operation O if applying O to any set in S produces another set in S.

```
∀O ∈ Operations, ∀S ∈ S: O(S) ∈ S
```

This property ensures that operations on ReactiveEntitySet produce predictable, well-defined results.

### Operations on ReactiveEntitySet

| Operation | Input | Output | Maintains closure |
|-----------|-------|--------|-------------------|
| Register | S, (id, data) | S ∪ {(id, data)} | Yes |
| Unregister | S, id | S \ {(id, _)} | Yes |
| SetData | S, id, data' | S with updated data | Yes |
| Filter (View) | S, predicate | S' ⊆ S | Yes |

All operations maintain closure. The result is always a valid ReactiveEntitySet (or subset thereof).

---

## View theory

Views are subsets of a ReactiveEntitySet defined by a predicate. There are two fundamental types.

### Static View R(c₀)

A static view depends only on the entity data.

```
R(c₀) = { (id, data) ∈ S | P(data) = true }
```

Where P is a predicate function: `P: TData → bool`

**Example**

```csharp
// All enemies with health below 30
var lowHealthView = enemies.CreateView(state => state.Health < 30);
```

**Characteristics**

- Predicate is fixed at creation time
- View membership updates automatically when entity data changes
- No external context required

### Dynamic View R(c₁)

A dynamic view depends on both entity data and an external context.

```
R(c₁) = { (id, data) ∈ S | P(data, context) = true }
```

Where P is a predicate function: `P: (TData, TContext) → bool`

**Example**

```csharp
// All enemies within 10 units of a given position
var inRangeView = enemies.CreateView<Vector3>(
    (state, position) => Vector3.Distance(state.Position, position) < 10f
);

// Evaluate with specific context
var nearbyEnemies = inRangeView.Evaluate(playerPosition);
```

**Characteristics**

- Predicate requires context to evaluate
- Entity data changes trigger automatic re-evaluation
- Context changes require explicit re-evaluation

### The "Reactive" in Views

Both R(c₀) and R(c₁) are **Reactive** with respect to entity state changes.

```
Entity state change → Predicate evaluation → View membership update → Notification
```

The difference is not about reactivity, but about whether the predicate requires external context.

---

## Structural closure

### Definition

Structural closure ensures that operations on a View cannot "escape" to the parent set.

```
View V = σ[P](S)

V is deterministically a subset of S
Operations on V stay within V
No path exists to access elements outside V
```

### Safe API design

This principle guides API design.

```csharp
var view = enemies.CreateView(state => state.Health < 30);

// ✅ Access within view
view.EntityIds
view.Count
view.ForEach(...)

// ❌ No backflow to parent set (API not provided)
// view.ParentSet  ← This should not exist
```

By not exposing a path back to the parent set, code operating on a View cannot accidentally access entities outside its scope.

---

## Performance implications

### Traditional approach

```
Query → Traverse all entities → Filter → Result
Cost: O(n) per query
```

### Reactive View approach

```
Entity state change
  → Evaluate view predicate
  → Add/remove from view
  → Notify subscribers

Cost: O(v) per change, where v = number of views
```

### When Reactive Views win

Reactive Views are more efficient when:

```
Change rate × View count < Entity count × Query frequency
```

**Good fit**

- Read frequency >> write frequency
- Same filter condition used repeatedly
- Real-time UI monitoring

**Poor fit**

- Write frequency >> read frequency
- Filter condition changes every time
- Very large number of views

---

## Formal definitions

For those interested in the precise mathematical formulation.

### ReactiveEntitySet

```
ReactiveEntitySet<TData> = (S, Σ, δ, notify)

Where:
  S: Set of (id, data) pairs
  Σ: Set of operations {Register, Unregister, SetData}
  δ: S × Σ → S (transition function)
  notify: S × Σ → Events (notification function)
```

### Static View R(c₀)

```
R(c₀) = σ[P](S) = { e ∈ S | P(e.data) }

Where:
  P: TData → bool (predicate)
  σ: Selection operator
```

### Dynamic View R(c₁)

```
R(c₁)(ctx) = σ[P(_, ctx)](S) = { e ∈ S | P(e.data, ctx) }

Where:
  P: (TData, TContext) → bool (parameterized predicate)
  ctx: TContext (evaluation context)
```

---

## Summary

| Concept | Description |
|---------|-------------|
| Set semantics | RES is a mathematical set with unique IDs |
| Closure | All operations produce valid sets |
| Static View R(c₀) | Filter by data only |
| Dynamic View R(c₁) | Filter by data + context |
| Reactive property | Views auto-update on entity changes |
| Structural closure | Views cannot escape to parent set |

---

## Implementation status

{: .note }
> Views are currently in the design phase and not yet implemented. The concepts described here represent the planned architecture.

---

## Next steps

- [Reactive Entity Sets Guide]({{ '/en/guides/reactive-entity-sets/' | relative_url }}) - Implementation details
- [Design Approach](approach) - Review the high-level philosophy
