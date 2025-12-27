---
layout: default
title: Getting Started
nav_order: 2
---

# Getting Started

This guide walks you through installing and using Reactive SO in your Unity project.

---

## Installation

### Via Git URL (Recommended)

1. Open **Window > Package Manager**
2. Click **+** > **Add package from git URL...**
3. Enter:
   ```
   https://github.com/tang3cko/EventChannels.git?path=/Assets/Event%20Channels
   ```
4. Click **Add**

### Via package.json

Add to your `Packages/manifest.json`:

```json
{
  "dependencies": {
    "com.tang3cko.reactiveso": "https://github.com/tang3cko/EventChannels.git?path=/Assets/Event%20Channels"
  }
}
```

---

## Quick Start: Your First Event Channel

### Step 1: Create an Event Channel Asset

Right-click in the Project window:

```
Create > Reactive SO > Channels > Void Event
```

Name it `OnPlayerDeath`.

### Step 2: Raise the Event (Publisher)

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

### Step 3: Subscribe to the Event (Subscriber)

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

### Step 4: Connect in Inspector

1. Select the Player GameObject
2. Drag the `OnPlayerDeath` asset to the serialized field
3. Do the same for GameManager

That's it! The Player and GameManager are now decoupled.

---

## Available Event Types

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

See [Event Types Reference](reference/event-types) for details.

---

## Best Practices

### Always Unsubscribe

Prevent memory leaks by unsubscribing in `OnDisable`:

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

### Use Null-Conditional Operator

Avoid null reference exceptions:

```csharp
// Good
onPlayerDeath?.RaiseEvent();

// Bad - throws if not assigned
onPlayerDeath.RaiseEvent();
```

### Assign in Inspector

Keep event flow visible by assigning channels in the Inspector rather than finding them in code.

---

## What's Next?

| If you want to... | Read... |
|-------------------|---------|
| Learn about all event types | [Event Types Reference](reference/event-types) |
| Share state between systems | [Variables Guide](guides/variables) |
| Track object collections | [Runtime Sets Guide](guides/runtime-sets) |
| Manage entity state | [Reactive Entity Sets Guide](guides/reactive-entity-sets) |
| Debug event flow | [Debugging Overview](debugging/overview) |
| Understand architecture | [Event Channels Guide](guides/event-channels) |
