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
| `Execute(...)` | `void` | Execute the action (abstract, must override) |

### Editor-only properties

| Property | Type | Description |
|----------|------|-------------|
| showInMonitor | `bool` | Show in Monitor Window during Play Mode |
| showInConsole | `bool` | Log executions to Console |

### Editor-only methods

| Method | Returns | Description |
|--------|---------|-------------|
| `NotifyActionExecuted(CallerInfo)` | `void` | Notify Monitor Window of execution |
| `LogAction(string)` | `void` | Log action execution to Console |

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
| `Execute(T value, ...)` | `void` | Execute with parameter (abstract, must override) |
| `Execute(...)` | `void` | Execute with default value (calls `Execute(default)`) |

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

    public override void Execute(
        string callerMember = "",
        string callerFile = "",
        int callerLine = 0)
    {
        AudioSource.PlayClipAtPoint(clip, Vector3.zero, volume);

#if UNITY_EDITOR
        var callerInfo = new CallerInfo(callerMember, callerFile, callerLine);
        NotifyActionExecuted(callerInfo);
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

    public override void Execute(Vector3 position,
        string callerMember = "",
        string callerFile = "",
        int callerLine = 0)
    {
        Object.Instantiate(prefab, position, Quaternion.identity);

#if UNITY_EDITOR
        var callerInfo = new CallerInfo(callerMember, callerFile, callerLine);
        NotifyActionExecuted(callerInfo);
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
2. Call `LogAction(string)` in your Execute method
3. Messages appear in Console during Play Mode

### Custom log messages

```csharp
public override void Execute(...)
{
    // Your logic here

#if UNITY_EDITOR
    NotifyActionExecuted(new CallerInfo(callerMember, callerFile, callerLine));
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

    public override void Execute(
        string callerMember = "",
        string callerFile = "",
        int callerLine = 0)
    {
        foreach (var action in actions)
        {
            action?.Execute();
        }

#if UNITY_EDITOR
        NotifyActionExecuted(new CallerInfo(callerMember, callerFile, callerLine));
#endif
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

    public override void Execute(
        string callerMember = "",
        string callerFile = "",
        int callerLine = 0)
    {
        if (condition != null && condition.Value)
        {
            trueAction?.Execute();
        }
        else
        {
            falseAction?.Execute();
        }

#if UNITY_EDITOR
        NotifyActionExecuted(new CallerInfo(callerMember, callerFile, callerLine));
#endif
    }
}
```

### Random action

```csharp
public class RandomAction : ActionSO
{
    [SerializeField] private ActionSO[] actions;

    public override void Execute(
        string callerMember = "",
        string callerFile = "",
        int callerLine = 0)
    {
        if (actions.Length > 0)
        {
            int index = Random.Range(0, actions.Length);
            actions[index]?.Execute();
        }

#if UNITY_EDITOR
        NotifyActionExecuted(new CallerInfo(callerMember, callerFile, callerLine));
#endif
    }
}
```

---

## Best practices

### Always wrap editor code

```csharp
public override void Execute(...)
{
    // Runtime logic here

#if UNITY_EDITOR
    // Monitoring and logging only in editor
    NotifyActionExecuted(new CallerInfo(callerMember, callerFile, callerLine));
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
