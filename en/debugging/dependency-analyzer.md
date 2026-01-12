---
layout: default
title: Dependency analyzer
parent: Debugging
nav_order: 3
---

# Dependency analyzer

## Purpose

This guide explains how to use the Dependency Analyzer for static analysis. You will learn to find unused event channels, detect unassigned fields, and understand refactoring impact.

---

## Opening the window

Navigate to **Window > Reactive SO > Dependency Analyzer**.

![Dependency Analyzer]({{ '/assets/images/debugging/dependency-analyzer.png' | relative_url }})

---

## Tabs

The Dependency Analyzer has four tabs.

| Tab | Description |
|-----|-------------|
| Event Channels | Analyze event channel usage and unassigned fields |
| Actions | Analyze action usage and unassigned fields |
| Variables | Analyze variable usage and unassigned fields |
| Runtime Sets | Analyze runtime set usage and unassigned fields |

All tabs provide the same functionality for their respective asset types.

---

## Use cases

### Detecting unassigned fields

**Problem**: You added a PlayerController but forgot to assign the event channel. The game crashes at runtime with NullReferenceException.

**Steps**:

1. Open Dependency Analyzer
2. Click **Scan Project**
3. Check the Summary line for unassigned fields
4. Review the Unassigned Event Channel Fields section
5. Click the arrow to jump to each location

**Example**:

```text
Summary: 20 Event Channels, 1 unused, 3 unassigned fields

Unassigned Event Channel Fields:
  MainScene.unity > Player.PlayerController.onPlayerDeath
  EnemyPrefab.prefab > Enemy.EnemyAI.spawnEvent
  UICanvas.prefab > HealthBar.onHealthChanged
```

### Finding unused event channels

**Problem**: Your project has 50 event channels and you do not know which are unused.

**Steps**:

1. Open Dependency Analyzer
2. Click **Scan Project**
3. Enable **Show Unused Only**
4. Review the list

You can safely delete event channels with zero usages.

### Checking refactoring impact

**Problem**: Before deleting `OnPlayerDeath`, you need to know where it is used.

**Steps**:

1. Open Dependency Analyzer
2. Click **Scan Project**
3. Search for "OnPlayerDeath"
4. Expand the event channel to see all usages
5. Click the arrow to jump to each location

**Example**:

```text
OnPlayerDeath (VoidEventChannelSO) - 5 usage(s)
  MainScene.unity
    Path: Assets/Scenes/MainScene.unity
    GameObject: Player
    Component: PlayerController
    Field: onPlayerDeath
  BossController.prefab
  GameOverUI.prefab
```

### Documenting usage

**Problem**: Team members ask "Where is this event channel used?" repeatedly.

**Steps**:

1. Open Dependency Analyzer
2. Click **Scan Project**
3. Click **Export** > Export to Markdown
4. Save the report
5. Share with team or commit to version control

---

## Window features

### Summary line

```text
Summary: 15 Event Channels, 2 unused, 3 unassigned fields
```

| Stat | Description |
|------|-------------|
| Event Channels | Total count in project |
| unused | Channels with no references |
| unassigned fields | Fields declared but not assigned |

### Tree view

Event channels display in a tree structure:

```text
OnPlayerDeath (VoidEventChannelSO) - 3 usage(s)
  MainScene.unity
    Path: Assets/Scenes/MainScene.unity
    GameObject: Player
    Component: PlayerController
    Field: onPlayerDeath
  BossController.prefab
  GameOverUI.prefab
```

Click the arrow to jump to assets.

### Unassigned fields section

```text
Unassigned Event Channel Fields:

  MainScene.unity > Player.PlayerController.onPlayerDeath
  EnemyPrefab.prefab > Enemy.EnemyAI.spawnEvent
```

Click the arrow to jump and fix.

---

## Search and filter

Use the search bar to filter:

- By name: `OnPlayerDeath`
- By type: `Void`, `Int`, `Float`
- Partial matches: `Player` matches `OnPlayerDeath`, `OnPlayerJumped`

### Show Unused Only

Enable to display only event channels with zero references. Use this to identify deletable assets.

---

## Exporting results

Click **Export** and select a format:

| Format | Use Case |
|--------|----------|
| Markdown | Human-readable documentation |
| JSON | Programmatic analysis |
| CSV | Spreadsheet analysis |

### Markdown example

```markdown
# Event Channels Dependency Report

Generated: 2025-12-27 10:30:00

## Summary

- Total Event Channels: 15
- Used Event Channels: 13
- Unused Event Channels: 2

### OnPlayerSpawn (VoidEventChannelSO)
**Path:** Assets/EventChannels/OnPlayerSpawn.asset
**Usages:** 3

1. **Assets/Scenes/MainScene.unity**
   - GameObject: SpawnManager
   - Component: SpawnManager
   - Field: spawnEventChannel
```

### JSON example

```json
{
  "GeneratedAt": "2025-12-27T10:30:00Z",
  "Summary": {
    "TotalEventChannels": 15,
    "UsedEventChannels": 13,
    "UnusedEventChannels": 2
  },
  "EventChannels": [
    {
      "Name": "OnPlayerSpawn",
      "Type": "VoidEventChannelSO",
      "UsageCount": 3,
      "Usages": [...]
    }
  ]
}
```

### CSV example

```csv
EventChannelName,EventChannelType,AssetPath,UsageCount,IsUsed
OnPlayerSpawn,VoidEventChannelSO,Assets/EventChannels/OnPlayerSpawn.asset,3,TRUE
OnPlayerDeath,VoidEventChannelSO,Assets/EventChannels/OnPlayerDeath.asset,0,FALSE
```

---

## Limitations

The Dependency Analyzer uses static analysis only. It does not detect:

- Dynamic references (`Resources.Load`, `Addressables.LoadAssetAsync`)
- String-based references
- Runtime-created references

Use [Event Monitor](monitor) during Play Mode to detect runtime references.

---

## Performance

Scan times depend on project size:

| Assets | Time |
|--------|------|
| 100 | ~5 seconds |
| 500 | ~15 seconds |
| 1000 | ~30 seconds |

A progress bar shows scan status. You can cancel if needed.

---

## Tips

- Run before major refactoring
- Export to Markdown and commit to version control
- Use Show Unused Only periodically to keep the project clean
- Combine with Event Monitor for both static and dynamic analysis

---

## References

- [Debugging Overview]({{ '/en/debugging/' | relative_url }}) - All debugging tools
- [Event Monitor](monitor) - Runtime event tracking
- [Troubleshooting]({{ '/en/troubleshooting' | relative_url }}) - Common issues