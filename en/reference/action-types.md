---
layout: default
title: Action types
parent: Reference
nav_order: 5
---

# Action types

{: .note }
> Actions are available since v2.1.0.

## Purpose

This reference documents the Action API for creating data-driven, reusable commands. You will find the API reference, implementation patterns, and examples for the ActionSO base classes.

---

## Type overview

| Type | Description |
|------|-------------|
| ActionSO | Base class for parameterless actions |
| ActionSO&lt;T&gt; | Generic base class for actions with a parameter |

---

## ActionSO

Abstract base class for ScriptableObject-based actions implementing the Command pattern.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| Description | `string` | User-defined description shown in Inspector |

### Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `Execute(...)` | `void` | Execute the action (non-virtual; centralizes caller-info capture and Monitor notification, then calls `OnExecute()` internally) |
| `OnExecute(...)` | `void` | Implements the action's actual behavior (abstract, must override) |

### Editor-only properties

| Property | Type | Description |
|----------|------|-------------|
| showInMonitor | `bool` | Show in Monitor Window during Play Mode |
| showInConsole | `bool` | Log executions to Console |

### Editor-only methods

| Method | Returns | Description |
|--------|---------|-------------|
| `LogAction(string)` | `void` | Log action execution to Console (call from within `OnExecute()`) |

### Editor-only events

| Event | Type | Description |
|-------|------|-------------|
| OnAnyActionExecuted | `Action<ActionSO, CallerInfo>` | Static event for monitoring |

---

## ActionSO&lt;T&gt;

Generic base class for actions that accept a parameter at execution time.

### Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `Execute(T value, ...)` | `void` | Execute with parameter (non-virtual; calls `OnExecute(T value)` internally) |
| `OnExecute(T value)` | `void` | Implements the actual behavior with a parameter (abstract, must override) |
| `Execute(...)` | `void` | Execute with default value (calls `OnExecute(default)` internally) |

---

## Implementing ActionSO

Basic implementation without parameters.

### Create

```csharp
using Tang3cko.ReactiveSO;
using UnityEngine;

[CreateAssetMenu(
    fileName = "PlaySound",
    menuName = "Game/Actions/Play Sound"
)]
public class PlaySoundAction : ActionSO
{
    [Header("Settings")]
    [SerializeField] private AudioClip clip;
    [SerializeField] private float volume = 1f;

    protected override void OnExecute()
    {
        AudioSource.PlayClipAtPoint(clip, Vector3.zero, volume);

#if UNITY_EDITOR
        LogAction($"Played {clip.name}");
#endif
    }
}
```

### Usage

```csharp
[SerializeField] private ActionSO playSoundAction;

public void OnButtonClick()
{
    playSoundAction?.Execute();
}
```

---

## Implementing ActionSO&lt;T&gt;

Implementation with a parameter.

### Create

```csharp
using Tang3cko.ReactiveSO;
using UnityEngine;

[CreateAssetMenu(
    fileName = "SpawnAtPosition",
    menuName = "Game/Actions/Spawn At Position"
)]
public class SpawnAtPositionAction : ActionSO<Vector3>
{
    [Header("Settings")]
    [SerializeField] private GameObject prefab;

    protected override void OnExecute(Vector3 position)
    {
        Object.Instantiate(prefab, position, Quaternion.identity);

#if UNITY_EDITOR
        LogAction($"Spawned at {position}");
#endif
    }
}
```

### Usage

```csharp
[SerializeField] private ActionSO<Vector3> spawnAction;

public void SpawnEnemy()
{
    Vector3 spawnPoint = GetRandomSpawnPoint();
    spawnAction?.Execute(spawnPoint);
}
```

---

## Caller information

Actions automatically track caller information for debugging.

### CallerInfo structure

| Field | Type | Description |
|-------|------|-------------|
| MemberName | `string` | Name of calling method |
| FilePath | `string` | Full path to calling file |
| LineNumber | `int` | Line number of call |

### Formatted output

```csharp
// CallerInfo.ToString() returns:
// "FileName.cs:MethodName:42"
```

### Important notes

- Do not pass explicit values to caller parameters
- Let the compiler fill them automatically
- Values are only meaningful when left as defaults

```csharp
// Good: Let compiler fill caller info
action.Execute();

// Bad: Explicit values lose caller tracking
action.Execute("", "", 0);
```

---

## Monitor integration

Actions integrate with the Monitor Window for real-time debugging.

### Enable monitoring

1. Select the action asset in Project window
2. In Inspector, enable `Show In Monitor`
3. Open Monitor Window (Window > Reactive SO > Monitor)
4. Enter Play Mode and trigger the action

### Console logging

1. Enable `Show In Console` in Inspector
2. Call `LogAction(string)` in your `OnExecute` method
3. Messages appear in Console during Play Mode

### Custom log messages

```csharp
protected override void OnExecute()
{
    // Your logic here

#if UNITY_EDITOR
    LogAction($"Custom message with {details}");
#endif
}
```

---

## Common patterns

### Multiple actions in sequence

```csharp
public class SequenceAction : ActionSO
{
    [SerializeField] private ActionSO[] actions;

    protected override void OnExecute()
    {
        foreach (var action in actions)
        {
            action?.Execute();
        }
    }
}
```

### Conditional action

```csharp
public class ConditionalAction : ActionSO
{
    [SerializeField] private BoolVariableSO condition;
    [SerializeField] private ActionSO trueAction;
    [SerializeField] private ActionSO falseAction;

    protected override void OnExecute()
    {
        if (condition != null && condition.Value)
        {
            trueAction?.Execute();
        }
        else
        {
            falseAction?.Execute();
        }
    }
}
```

### Random action

```csharp
public class RandomAction : ActionSO
{
    [SerializeField] private ActionSO[] actions;

    protected override void OnExecute()
    {
        if (actions.Length > 0)
        {
            int index = Random.Range(0, actions.Length);
            actions[index]?.Execute();
        }
    }
}
```

---

## Best practices

### Always wrap editor code

```csharp
protected override void OnExecute()
{
    // Runtime logic here

#if UNITY_EDITOR
    // Logging only in editor (Monitor notification is handled by Execute() automatically)
    LogAction("Details");
#endif
}
```

### Use null-conditional operator

```csharp
// Safe execution
action?.Execute();
```

### Keep actions focused

One action should do one thing. Compose complex behaviors from simple actions.

### Document with description field

Set the `description` field in Inspector to explain what the action does.

---

## References

- [Actions Guide]({{ '/en/guides/actions' | relative_url }}) - How to use actions
- [Monitor Window]({{ '/en/debugging/monitor' | relative_url }}) - Debug action execution
- [Debugging Overview]({{ '/en/debugging/' | relative_url }}) - All debugging tools
