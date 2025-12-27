---
layout: default
title: Home
nav_order: 1
---

# Reactive SO

ScriptableObject-based reactive architecture for Unity with Event Channels, Variables, Runtime Sets, and Reactive Entity Sets.

---

## Why Reactive SO?

Traditional Unity development often leads to tightly coupled components that are hard to test and maintain. Reactive SO solves this by providing:

- **Complete Decoupling** - Publishers and subscribers communicate through ScriptableObject assets without direct references
- **Visual Event Flow** - See event dependencies directly in the Unity Inspector
- **Zero Runtime Overhead** - Debugging features only active in Editor
- **Strongly-Typed** - 12 event types covering all common Unity types

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
entitySet.AddOrUpdate(entityId, new EntityData { ... });
```

[Learn more about Reactive Entity Sets]({{ '/en/guides/reactive-entity-sets' | relative_url }})

---

## Getting Started

Ready to use Reactive SO in your project?

[Get Started]({{ '/en/getting-started' | relative_url }}){: .btn .btn-primary }

---

## Requirements

- Unity 2022.3 or newer
- .NET Standard 2.1

---

## Links

- [GitHub Repository](https://github.com/tang3cko/EventChannels)
- [Asset Store](https://assetstore.unity.com/) (coming soon)
- [日本語ドキュメント]({{ '/ja/' | relative_url }})
