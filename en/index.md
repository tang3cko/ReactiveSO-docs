---
layout: default
title: Home
nav_order: 1
---

# Reactive SO

ScriptableObject-based reactive architecture for Unity with Event Channels, Variables, Runtime Sets, and Reactive Entity Sets.

---

## Purpose

This documentation covers everything you need to use Reactive SO in your Unity projects. You will learn how to install, configure, and use each feature effectively.

---

## Why Reactive SO?

Traditional Unity development often leads to tightly coupled components that are hard to test and maintain. Reactive SO addresses these challenges through four core principles.

### Decoupled by Design

Publishers and subscribers communicate through ScriptableObject assets without direct references. No singletons, no FindObjectOfType, no tight coupling.

### Inspector-First

Wire up event connections, configure shared state, and verify dependencies—all from the Inspector. Your architecture becomes visible and editable without touching code.

### Observable

See what's happening at runtime. The Monitor Window shows events firing, variables changing, and sets updating in real-time. Asset Browser and Dependency Analyzer help you understand and maintain your project.

### Scalable

Start with Event Channels for simple notifications. Add Variables for shared state, Runtime Sets for object tracking. Scale up to Reactive Entity Sets for tens of thousands of entities with Job System integration.

---

## Core Features

### Event Channels

Global event notifications using ScriptableObject assets.

```csharp
// Raise an event
onPlayerDeath?.RaiseEvent();

// Subscribe to an event
onPlayerDeath.OnEventRaised += HandlePlayerDeath;
```

[Learn more about Event Channels]({{ '/en/guides/event-channels' | relative_url }})

### Variables

Reactive shared state with automatic change detection.

```csharp
// Variable changes trigger events automatically
playerHealth.Value = 100;
playerHealth.OnValueChanged += UpdateHealthUI;
```

[Learn more about Variables]({{ '/en/guides/variables' | relative_url }})

### Runtime Sets

Manage dynamic object collections without singletons.

```csharp
// Track all enemies in scene
enemySet.Add(this.gameObject);
foreach (var enemy in enemySet.Items) { ... }
```

[Learn more about Runtime Sets]({{ '/en/guides/runtime-sets' | relative_url }})

### Reactive Entity Sets

Centralized, scene-independent entity state management.

```csharp
// Scene-independent state with O(1) access
entitySet.Register(entityId, new EntityData { ... });
```

[Learn more about Reactive Entity Sets]({{ '/en/guides/reactive-entity-sets' | relative_url }})

---

## Additional Features

### ActionSO

Command pattern implemented as ScriptableObject assets. Define reusable actions, wire them in the Inspector, and execute them manually during Play Mode for testing.

[Learn more about Actions]({{ '/en/guides/actions' | relative_url }})

### R3 Integration

When [R3](https://github.com/Cysharp/R3) is installed, Event Channels and Reactive Entity Sets can be converted to Observable streams for reactive programming workflows.

[Learn more about Integrations]({{ '/en/integrations/' | relative_url }})

---

## Getting Started

Ready to use Reactive SO in your project?

[Get Started]({{ '/en/getting-started' | relative_url }}){: .btn .btn-primary }

---

## Requirements

- **v2.1.0+**: Unity 6000.0 (Unity 6.0) or newer
- v1.1.1–v2.0.0: Unity 6000.2 (Unity 6.2) or newer
- v1.0.0–v1.1.0: Unity 2022.3 or newer

---

## Links

- [Asset Store](https://assetstore.unity.com/packages/tools/game-toolkits/reactive-so-339926)
- [Blog Series](https://tang3cko.hashnode.dev/series/reactive-so)
- [日本語ドキュメント]({{ '/ja/' | relative_url }})
