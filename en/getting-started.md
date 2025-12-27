---
layout: default
title: Getting Started
nav_order: 2
---

# Getting Started

---

## Purpose

This guide walks you through installing Reactive SO and creating your first Event Channel. You will learn the basic workflow for decoupling game systems.

---

## Installation

Reactive SO is installed via Unity Package Manager.

1. Open **Window > Package Manager**
2. Find **Reactive SO** in the package list
3. Click **Install**

If you purchased from the [Asset Store](https://assetstore.unity.com/packages/tools/game-toolkits/reactive-so-339926), use **My Assets** tab to import the package.

---

## Quick start: your first event channel

### Step 1: Create an event channel asset

Right-click in the Project window.

```text
Create > Reactive SO > Channels > Void Event
```

<!-- TODO: Add screenshot of Create menu showing Reactive SO > Channels submenu -->

Name it `OnPlayerDeath`.

### Step 2: Raise the event (publisher)

```csharp
using Tang3cko.ReactiveSO;
using UnityEngine;

public class Player : MonoBehaviour
{
    [SerializeField] private VoidEventChannelSO onPlayerDeath;

    public void Die()
    {
        onPlayerDeath?.RaiseEvent();
    }
}
```

### Step 3: Subscribe to the event (subscriber)

```csharp
using Tang3cko.ReactiveSO;
using UnityEngine;

public class GameManager : MonoBehaviour
{
    [SerializeField] private VoidEventChannelSO onPlayerDeath;

    private void OnEnable()
    {
        onPlayerDeath.OnEventRaised += HandlePlayerDeath;
    }

    private void OnDisable()
    {
        onPlayerDeath.OnEventRaised -= HandlePlayerDeath;
    }

    private void HandlePlayerDeath()
    {
        Debug.Log("Game Over!");
    }
}
```

### Step 4: Connect in the Inspector

1. Select the Player GameObject
2. Drag the `OnPlayerDeath` asset to the serialized field
3. Do the same for GameManager

<!-- TODO: Add screenshot showing Inspector with Event Channel field assignment -->

That's it! The Player and GameManager are now decoupled.

---

## Available event types

| Type | Use Case | Example |
|------|----------|---------|
| Void | Notifications without data | OnGameStart, OnPlayerDeath |
| Int | Integer values | OnScoreChanged, OnLevelUp |
| Float | Float values | OnHealthChanged, OnProgress |
| Bool | Boolean states | OnPaused, OnMuted |
| String | Text messages | OnDialogue, OnNotification |
| Vector2 | 2D positions/directions | OnInputAxis, OnTouchPosition |
| Vector3 | 3D positions/directions | OnSpawnPosition, OnTargetPosition |
| Quaternion | Rotations | OnCameraRotation |
| Color | Colors | OnThemeChanged |
| GameObject | Object references | OnEnemySpawned, OnTargetChanged |
| Long | Large integers | OnTimestamp |
| Double | Precise decimals | OnPreciseValue |

See [Event Types Reference]({{ '/en/reference/event-types' | relative_url }}) for details.

---

## Best practices

### Always unsubscribe

Prevent memory leaks by unsubscribing in `OnDisable`.

```csharp
private void OnEnable()
{
    eventChannel.OnEventRaised += HandleEvent;
}

private void OnDisable()
{
    eventChannel.OnEventRaised -= HandleEvent;  // Required!
}
```

### Use null-conditional operator

Avoid null reference exceptions.

```csharp
// Good
onPlayerDeath?.RaiseEvent();

// Bad - throws if not assigned
onPlayerDeath.RaiseEvent();
```

### Assign in the Inspector

Keep event flow visible by assigning channels in the Inspector rather than finding them in code.

---

## What's next?

| If you want to... | Read... |
|-------------------|---------|
| Learn about all event types | [Event Types Reference]({{ '/en/reference/event-types' | relative_url }}) |
| Share state between systems | [Variables Guide]({{ '/en/guides/variables' | relative_url }}) |
| Track object collections | [Runtime Sets Guide]({{ '/en/guides/runtime-sets' | relative_url }}) |
| Manage entity state | [Reactive Entity Sets Guide]({{ '/en/guides/reactive-entity-sets' | relative_url }}) |
| Debug event flow | [Debugging]({{ '/en/debugging/' | relative_url }}) |
| Understand architecture | [Event Channels Guide]({{ '/en/guides/event-channels' | relative_url }}) |
