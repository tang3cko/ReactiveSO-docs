---
layout: default
title: RES Design
parent: Design Philosophy
nav_order: 3
has_children: true
---

# Reactive Entity Sets: Design Philosophy

---

## Purpose

This section explains the design philosophy behind Reactive Entity Sets (RES). Understanding these concepts will help you decide when and how to use RES effectively.

RES is fundamentally different from other state management approaches in Unity. Before diving into implementation, it is important to understand **why** RES exists and what problems it solves.

---

## What you will learn

| Page | Topics |
|------|--------|
| [Design Approach](approach) | How RES differs from GPU Instancer and ECS, what RES prioritizes |
| [Entity, Object, and View](entity-object-view) | Entity vs Object distinction, GameObjects as Views, scene-independent data |
| [Data Guidelines](data-guidelines) | What data belongs in RES, data/logic separation pattern |
| [Set Theory Foundation](set-theory) | Mathematical basis, View types, correspondence with databases |

---

## Core insight

RES is built on a key architectural inversion.

**Traditional Unity pattern**

```
GameObject owns its data
  └── MonoBehaviour holds state
      └── Destroyed with GameObject
```

**RES pattern**

```
ScriptableObject owns the data (persistent)
  └── GameObject displays the data (view)
      └── Can be destroyed without losing state
```

This inversion enables cross-scene persistence, network synchronization, and clean separation between data and visualization.

---

## Quick navigation

**New to RES?**

Start with [Design Approach](approach) to understand how RES differs from other solutions.

**Want practical guidance?**

Jump to [Data Guidelines](data-guidelines) for rules on what data to include.

**Interested in theory?**

Explore [Set Theory Foundation](set-theory) for the mathematical basis.

---

## Next steps

After understanding the design philosophy, see the [Reactive Entity Sets Guide]({{ '/en/guides/reactive-entity-sets/' | relative_url }}) for practical implementation.
