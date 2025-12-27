---
layout: default
title: Event monitor
parent: Debugging
nav_order: 2
---

# Event monitor

## Purpose

This guide explains how to use the Event Monitor Window for real-time event tracking. You will learn to identify event flow issues, find duplicate events, and verify subscriber counts.

---

## Opening the window

Navigate to **Window > Reactive SO > Event Monitor**.

<!-- TODO: Add screenshot of Event Monitor Window showing event log with Time, Event Name, Type, Value, #L, and Caller columns -->

---

## Filtering events

Only event channels with **Show In Event Log** enabled appear in this window. New event channels have this enabled by default.

To disable logging for an event:

1. Select the event channel asset
2. In the Inspector, find the Debug section
3. Disable "Show In Event Log"

This reduces noise from high-frequency events you are not debugging.

---

## Window columns

| Column | Description |
|--------|-------------|
| Time | Seconds since Play Mode started |
| Event Name | Name of the event channel asset |
| Type | Event type (Void, Int, Float, etc.) |
| Value | Value passed with the event |
| #L | Number of active listeners |
| Caller | Code location that raised the event |

---

## Controls

| Control | Description |
|---------|-------------|
| Search | Filter events by name or type |
| Pause/Resume | Pause logging to examine events |
| Auto-scroll | Scroll to new events automatically |
| Export CSV | Save logs for external analysis |
| Clear | Remove all logged events |
| Max | Maximum log entries (100-10K) |

---

## Understanding caller information

The Caller column shows which code raised each event:

```
FileName.cs:MethodName:LineNumber
```

Example:

```
PlayerController.cs:TakeDamage:142
```

This tells you:

- **File**: PlayerController.cs
- **Method**: TakeDamage()
- **Line**: 142

Click the caller to open the file in your IDE (if supported).

### How it works

Caller information uses C# `CallerInfo` attributes at compile-time. This means zero runtime overhead.

### Limitation: UnityEvents

Caller info does not work when events are raised via UnityEvent (e.g., Button.OnClick in Inspector). The Caller column shows "-".

**Workaround**: Create a wrapper method:

```csharp
// In Inspector: Button.OnClick -> OnButtonClick
public void OnButtonClick()
{
    onButtonPressed?.RaiseEvent();  // Caller: "OnButtonClick"
}
```

---

## Use cases

### Is this event firing?

When a button does not work:

1. Open Event Monitor
2. Click the button in your game
3. Check if the event appears

**Event appears**: Problem is in subscriber.
**Event does not appear**: Check publisher and `RaiseEvent()` call.

### Finding duplicate events

When a sound plays twice:

1. Open Event Monitor
2. Trigger player death once
3. Count entries in the log
4. Check Caller column:

```
Time | Event Name    | Caller
0.5s | OnPlayerDeath | EnemyAI.cs:Attack:142
0.5s | OnPlayerDeath | FallDetector.cs:OnTriggerEnter:28
```

Two scripts raised the same event simultaneously.

### Tracking event chains

Complex chains like OnPlayerDeath -> OnGameOver -> OnHighScoreSaved:

1. Open Event Monitor
2. Trigger player death
3. See the complete sequence:

```
Time | Event Name       | Caller
0.5s | OnPlayerDeath    | Player.cs:TakeDamage:89
0.6s | OnGameOver       | GameManager.cs:HandlePlayerDeath:45
0.7s | OnHighScoreSaved | HighScoreManager.cs:HandleGameOver:67
```

### Verifying listener count

When 3 systems should respond but only 2 do:

1. Open Event Monitor
2. Trigger the event
3. Check the #L column
4. If #L shows 2, one system forgot to subscribe

Use Subscribers List to find which one.

---

## Search and filter

Use the search box to filter events:

- Event name: `OnPlayerDeath`
- Event type: `Void`, `Int`, `Float`
- Partial matches: `Player` matches `OnPlayerDeath`, `OnPlayerJumped`

Examples:

- Type `Death` to see death-related events
- Type `Int` to see integer events
- Type `UI` to see UI-related events

---

## Pause and analysis

Click **Pause** to freeze the log:

- When events fire too quickly to read
- To compare multiple event entries
- To investigate specific sequences

Click **Resume** to continue logging.

---

## Log persistence

Event logs behave as follows:

| Action | Behavior |
|--------|----------|
| Enter Play Mode | Logs cleared |
| During Play Mode | Events recorded |
| Exit Play Mode | Logs preserved for review |
| Next Play Mode | Previous logs cleared |

You can review logs after exiting Play Mode.

---

## Exporting logs

Click **Export CSV** to save logs. The file includes:

- Time, Event Name, Type, Value, Listeners, Caller

The CSV file is named with a timestamp and saved with UTF-8 encoding for Excel compatibility.

Use exports for:

- Event frequency analysis
- Performance profiling
- Memory leak detection
- Playtest data analysis

---

## Configuration

### Maximum log entries

1. Find the Max dropdown in the toolbar
2. Select from: 100, 200, 500, 1K, 2K, 5K, 10K
3. Setting is saved automatically

Use higher values for:

- Long event sequences
- Extended debugging sessions
- Pattern analysis

Note: Higher values consume more memory.

---

## Performance

The Event Monitor has negligible performance impact:

- Logging only occurs in Editor (not builds)
- Efficient string formatting
- Automatic entry limit
- Can be closed when not needed

---

## Tips

- Keep the window open on a second monitor
- Use search to focus on specific events
- Pause before complex sequences
- Click caller to jump to source code
- Disable logging for high-frequency events

---

## References

- [Debugging Overview](overview) - All debugging tools
- [Dependency Analyzer](dependency-analyzer) - Static analysis
- [Troubleshooting]({{ '/en/troubleshooting' | relative_url }}) - Common issues
