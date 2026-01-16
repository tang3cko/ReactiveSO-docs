---
layout: default
title: Design Approach
parent: RES Design
grand_parent: Design Philosophy
nav_order: 1
---

# Design Approach

---

## Purpose

This page explains how Reactive Entity Sets (RES) differs from other entity management solutions like GPU Instancer and ECS. Understanding these differences will help you choose the right tool for your project.

---

## Different tools for different goals

RES, GPU Instancer, and ECS are all valid solutions, but they optimize for different things.

| Aspect | GPU Instancer / ECS | Reactive Entity Sets |
|--------|---------------------|----------------------|
| Primary goal | Maximum performance | Observability and debuggability |
| Optimization target | Every-frame processing | Change notification |
| Complexity | High | Low |
| Workflow change | Significant | Minimal |

This is not about which is "better." Each approach serves different project needs.

---

## When to consider each approach

### GPU Instancer

Best for rendering thousands of similar objects with minimal overhead.

- Grass, trees, rocks, particles
- Objects that don't need individual state tracking
- Performance-critical visual elements

### ECS (Entity Component System)

Best for projects that need maximum throughput on large entity counts.

- Thousands of entities processed every frame
- Teams willing to learn a new paradigm
- Projects where every millisecond matters

### Reactive Entity Sets

Best for projects that prioritize development speed and debuggability.

- Moderate entity counts (hundreds to low thousands)
- Teams wanting to keep the GameObject workflow
- Projects where observable state changes matter
- Rapid iteration and debugging are priorities

---

## What RES prioritizes

RES makes deliberate trade-offs to optimize for developer experience.

### Observability

Every state change is an event. You can always see what changed, when, and why.

```csharp
// Every change triggers an observable event
entitySet.UpdateData(enemyId, state => {
    state.Health -= damage;
    return state;
});
// → OnDataChanged fires
// → Per-entity subscribers notified
// → Inspector shows the change
```

### Debuggability

RES integrates deeply with Unity's editor and provides monitoring tools.

- See all entity states in the Table View during Play Mode
- Track which entities are registered
- Monitor event flow in real-time
- No black-box behavior

### Predictability

Data changes are guaranteed to be reflected and notified.

- No silent mutations
- No missed updates
- Clear cause-and-effect relationships

### Architectural enforcement

RES naturally leads to clean separation between data and logic.

```
State (struct)     : Data only
Calculation logic  : Pure functions
RES                : Storage and notification
GameObject         : Visualization
```

---

## The workflow advantage

One of RES's key benefits is preserving the familiar GameObject workflow.

### ECS workflow

```
Traditional MonoBehaviour → Convert to Entity/Component/System
                         → New mental model
                         → Different debugging tools
                         → Significant learning curve
```

### RES workflow

```
Traditional MonoBehaviour → Add RES integration
                         → Same mental model
                         → Same Inspector debugging
                         → Gradual adoption possible
```

You can adopt RES incrementally. Start with one system, see if it works for you, then expand.

---

## Performance characteristics

RES uses a Sparse Set data structure internally.

| Operation | Time Complexity |
|-----------|-----------------|
| Register | O(1) |
| Unregister | O(1) |
| GetData | O(1) |
| SetData | O(1) |
| Iteration | O(n) |

RES is efficient when change frequency is low relative to entity count.

**RES excels when**
- Less than 10% of entities change per frame
- Entity count is moderate (hundreds to low thousands)
- Multiple systems need to react to the same state changes

**Consider alternatives when**
- All entities update every frame
- Entity count is in the tens of thousands
- Raw performance is the top priority

---

## Summary

| Approach | Optimizes for | Best when |
|----------|---------------|-----------|
| GPU Instancer | Rendering performance | Many similar visual objects |
| ECS | Processing throughput | Massive entity counts, every-frame updates |
| **RES** | **Developer experience** | **Observable state, debuggability, gradual adoption** |

RES is not trying to replace GPU Instancer or ECS. It fills a different niche: projects where development speed, debuggability, and maintainability matter more than squeezing out every last frame.

---

## Next steps

- [Entity, Object, and View](entity-object-view) - Understand the core conceptual model
- [Data Guidelines](data-guidelines) - Learn what data belongs in RES
