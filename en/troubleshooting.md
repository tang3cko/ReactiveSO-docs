---
layout: default
title: Troubleshooting
nav_order: 7
---

# Troubleshooting

## Purpose

This page covers common issues, known limitations, and frequently asked questions about Reactive SO.

---

## Common issues

### Events not firing

**Symptom**: You call `RaiseEvent()` but nothing happens.

**Checklist**

1. **Is the event channel assigned?**
   - Check for `None` in the Inspector
   - Drag the correct ScriptableObject asset

2. **Are you using null-conditional operator?**
   ```csharp
   // Good
   eventChannel?.RaiseEvent();

   // Bad - throws NullReferenceException if not assigned
   eventChannel.RaiseEvent();
   ```

3. **Is anyone subscribed?**
   - Open Event Monitor and check #L column
   - Use Subscribers List to see listeners
   - Verify subscriber's GameObject is active

4. **Is the event actually raising?**
   - Open Event Monitor
   - If event appears, the problem is in subscribers
   - If event does not appear, check the publisher code

### Memory leaks / duplicate events

**Symptom**: Events fire multiple times or memory grows after scene transitions.

**Cause**: Forgot to unsubscribe in `OnDisable`.

**Fix**

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

**Diagnosis**: Use Subscribers List before and after scene transitions. If destroyed objects remain in the list, you have a leak.

### Events fire too early

**Symptom**: Subscriber's `Start()` has not run when the event fires.

**Solutions**

**Option 1** - Delay the event
```csharp
private IEnumerator Start()
{
    yield return null;  // Wait one frame
    onGameStarted?.RaiseEvent();
}
```

**Option 2** - Subscribe earlier
```csharp
private void Awake()
{
    eventChannel.OnEventRaised += HandleEvent;
}

private void OnDisable()
{
    eventChannel.OnEventRaised -= HandleEvent;
}
```

### Wrong event channel referenced

**Symptom**: Multiple event channels have similar names and you reference the wrong one.

**Fix** Use clear names and organize in folders:

```text
ScriptableObjects/Events/
├── Player/
│   ├── OnPlayerDeath.asset
│   └── OnPlayerHealthChanged.asset
├── UI/
│   └── OnMenuOpened.asset
└── Game/
    └── OnGamePaused.asset
```

### Subscribers not updating

**Symptom**: Event fires but subscriber does not react.

**Checklist**

1. Is the GameObject active?
2. Is the MonoBehaviour enabled?
3. Did `OnEnable()` run?
4. Is the subscription correct?
   ```csharp
   // Correct
   eventChannel.OnEventRaised += HandleEvent;

   // Wrong - overwrites other subscribers
   eventChannel.OnEventRaised = HandleEvent;
   ```
5. Does the method signature match?
   ```csharp
   // For IntEventChannelSO
   void HandleEvent(int value) { }  // Correct
   void HandleEvent() { }           // Wrong signature
   ```

---

## Known limitations

### Caller information with UnityEvents

Caller information only works for direct code calls. It does not work when events are raised via UnityEvent (e.g., Button.OnClick in Inspector).

**Works**
```csharp
public void OnButtonClick()
{
    onButtonPressed?.RaiseEvent();  // Caller: "OnButtonClick"
}
```

**Does not work**
```
Inspector: Button.OnClick → EventChannel.RaiseEvent
Caller shows: "-"
```

**Workaround**: Create a wrapper method instead of calling `RaiseEvent` directly from Inspector.

### Manual Trigger custom types

Manual Trigger only supports the 12 built-in event types. Custom types display a help message instead of a trigger button.

### Subscribers List scope

Subscribers List only shows MonoBehaviour subscribers. Other subscribers (ScriptableObjects, static classes) appear in the #L count but not in the list.

---

## FAQ

### Can I use Reactive SO in builds?

Yes. All features work in builds with zero overhead. Debugging tools (Event Monitor, Manual Trigger, Subscribers List, Caller Info) are Editor-only and automatically stripped from builds.

### What is the performance cost?

Nearly zero. Event channels use standard C# events internally. Debugging features are completely removed from builds.

### Can I use event channels with coroutines?

Yes.

```csharp
private void OnEnable()
{
    eventChannel.OnEventRaised += HandleEvent;
}

private void HandleEvent()
{
    StartCoroutine(HandleEventCoroutine());
}

private IEnumerator HandleEventCoroutine()
{
    // Your coroutine logic
}
```

### Can I pass multiple values in one event?

Not directly. Use one of these approaches.

**Option 1** - Use a struct
```csharp
[System.Serializable]
public struct PlayerDeathData
{
    public Vector3 position;
    public string killerName;
    public int score;
}

public class PlayerDeathEventChannelSO : EventChannelSO<PlayerDeathData> { }
```

**Option 2** - Use multiple events
```csharp
onPlayerDeath?.RaiseEvent();
onDeathPosition?.RaiseEvent(position);
onFinalScore?.RaiseEvent(score);
```

### Can I subscribe from ScriptableObjects?

Yes.

```csharp
public class GameSettings : ScriptableObject
{
    [SerializeField] private VoidEventChannelSO onGameStarted;

    private void OnEnable()
    {
        onGameStarted.OnEventRaised += HandleGameStarted;
    }

    private void OnDisable()
    {
        onGameStarted.OnEventRaised -= HandleGameStarted;
    }

    private void HandleGameStarted()
    {
        // Initialize settings
    }
}
```

Note: ScriptableObject subscribers do not appear in Subscribers List but are counted in the #L column.

### How do I unit test event channels?

```csharp
using NUnit.Framework;
using Tang3cko.ReactiveSO;

public class EventChannelTests
{
    [Test]
    public void EventRaisesCorrectly()
    {
        // Arrange
        var eventChannel = ScriptableObject.CreateInstance<IntEventChannelSO>();
        int receivedValue = 0;
        eventChannel.OnEventRaised += (value) => receivedValue = value;

        // Act
        eventChannel.RaiseEvent(42);

        // Assert
        Assert.AreEqual(42, receivedValue);
    }
}
```

### Can I use event channels for networking?

Event channels work locally within a single Unity instance. For networked games, consider the following approach.

- Use event channels for local game logic
- Use your networking solution for network communication
- Raise event channels in response to network events

Here is an example of raising local events from network callbacks.

```csharp
[ClientRpc]
void RpcPlayerDeath()
{
    // Runs on all clients
    onPlayerDeath?.RaiseEvent();  // Local event for UI/audio
}
```

---

## Still having issues?

If you are experiencing issues not covered here, try the following steps.

1. Check Event Monitor to verify event flow
2. Use Subscribers List to verify subscriptions
3. Try Manual Trigger to isolate publisher vs subscriber issues
4. Add `Debug.Log` in `OnEnable`, `OnDisable`, and handlers
5. Review [Getting Started]({{ '/en/getting-started' | relative_url }}) for basic setup

---

## Reporting issues

When reporting issues, include the following information.

- Unity version
- Reactive SO version
- Steps to reproduce
- Event Monitor output
- Subscribers List screenshot

For support, use the following channels.

- [Report issues on GitHub](https://github.com/tang3cko/ReactiveSO-docs/issues)
- [Asset Store page](https://assetstore.unity.com/packages/tools/game-toolkits/reactive-so-339926)
- [X (@tang3cko)](https://x.com/tang3cko)

---

## References

- [Event Channels Guide]({{ '/en/guides/event-channels' | relative_url }}) - Basic usage
- [Debugging]({{ '/en/debugging/' | relative_url }}) - Debugging tools
- [Event Monitor]({{ '/en/debugging/event-monitor' | relative_url }}) - Real-time tracking
