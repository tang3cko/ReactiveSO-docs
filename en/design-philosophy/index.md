---
layout: default
title: Design Philosophy
nav_order: 3
has_children: true
---

# Design Philosophy

---

## Purpose

This section explains the foundational concepts behind Reactive SO. Understanding these principles will help you make better architectural decisions and use each feature effectively.

---

## Core principle

Reactive SO is built on one fundamental idea: **ScriptableObjects are global, shared resources**.

This means:

- They persist across all scenes
- All scripts referencing the same asset share the same instance
- They are perfect for global state and cross-system communication

Understanding this principle is critical for choosing the right tool for each situation.

---

## What you will learn

| Page | Topics |
|------|--------|
| [ScriptableObject Basics](scriptableobject-basics) | Global shared resources, Entity vs Object, GameObjects as Views, scene persistence |
| [Architecture Patterns](architecture-patterns) | Four tools comparison, Instance vs Global rule, common patterns, decision guides |

---

## Quick navigation

**New to Reactive SO?**

Start with [Getting Started]({{ '/en/getting-started' | relative_url }}) for installation and basic usage, then return here for deeper understanding.

**Already using Reactive SO?**

Jump to [Architecture Patterns](architecture-patterns) for decision guides on choosing the right tool.
