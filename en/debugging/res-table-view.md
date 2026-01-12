---
layout: default
title: RES Table View
parent: Debugging
nav_order: 5
---

# RES Table View

{: .warning }
> ReactiveEntitySetSO is an experimental feature. The API may change in future versions.

## Purpose

This guide explains how to use RES Table View to view and edit ReactiveEntitySet entity data in a table format. You will learn how to capture snapshots while paused in Play Mode, inspect and edit individual entity values, and save/restore data using binary files.

---

## Opening the Window

Open the window using one of the following methods:

1. **From menu**: Window > Reactive SO > RES Table View
2. **From Inspector**: Select a ReactiveEntitySetSO asset and click "Open Table View"

![RES Table View]({{ '/assets/images/debugging/res-table-view.png' | relative_url }})

---

## Basic Usage

### Capturing a Snapshot

1. Enter Play Mode
2. Pause the game (Pause button)
3. Select the target ReactiveEntitySetSO in Target RES
4. Click "Capture Snapshot"

When capture succeeds, entity data is displayed in the table.

{: .note }
> Snapshots can only be captured while paused. Data is cleared when you resume.

### Editing Data

1. Double-click the cell you want to edit
2. Enter the new value
3. Press Enter to confirm, Escape to cancel

Edited values are immediately applied to the ReactiveEntitySet, firing the `OnDataChanged` event.

---

## Toolbar

| Element | Description |
|---------|-------------|
| Target RES | Select the target ReactiveEntitySetSO |
| Capture Snapshot | Capture current entity data |
| Save | Save captured data to a binary file |
| Load | Load data from a binary file, completely overwriting |

---

## Save / Load Feature

### Save

Saves captured snapshot data as a `.resdata` file.

- **Enabled when**: Captured data exists
- **File format**: Binary (.resdata)
- **Contents**: EntityID and all TData struct fields

### Load

{: .warning }
> Load completely overwrites current entity data. This operation cannot be undone.

Restores from a saved `.resdata` file, completely replacing the ReactiveEntitySet data.

- **Enabled when**: Paused and RES is selected
- **Behavior**: Calls `RestoreSnapshot()`, replacing all entities
- **Warning**: A confirmation dialog appears after file selection

### Binary Format

```
[Header] - 20 bytes
├─ Magic: "RES\0" (4 bytes)
├─ Version: int32
├─ TypeHash: int32 (hash of TData type)
├─ DataSize: int32 (size of TData struct)
└─ EntityCount: int32

[Data]
├─ EntityIds: int32[] (EntityCount items)
└─ EntityData: byte[] (EntityCount × DataSize bytes)
```

---

## Columns

| Column | Description |
|--------|-------------|
| Entity ID | Unique entity identifier (read-only) |
| (TData fields) | Each field of the TData struct is dynamically displayed as a column |

### Supported Types

| Type | Display Format | Editable |
|------|----------------|----------|
| int, float, byte, etc. | Number | ✓ |
| bool | True/False | ✓ (toggle) |
| Enum | Value name | ✓ (dropdown) |
| Vector2, Vector3 | (x, y, z) | × |
| Quaternion | Euler angles | × |
| Color | (r, g, b, a) | × |

{: .note }
> Compound types like Vector and Quaternion are display-only and cannot currently be edited.

---

## Use Cases

### Inspecting Entity States

Examine the current values of specific entities to identify the cause of issues. Pause during Play Mode and capture a snapshot to see all entity data at that point in time.

### Testing Game Logic

Test game logic behavior by directly editing values. For example, set an entity's HP to 0 to verify that death processing works correctly.

### Saving Bug Reproduction Data

Save the entity state at the moment a bug occurs as a `.resdata` file, then restore it later to continue debugging.

### Regression Test Baselines

Save specific states and load them during test execution to enable consistent testing under the same conditions.

---

## Limitations

- **Play Mode required**: Does not work in Edit Mode
- **Pause required**: Capture, edit, and load only work while paused
- **Type compatibility**: Loading a file with a different TData type shows a warning
- **Compound type editing**: Vector, Quaternion, Color cannot be edited
- **Large data**: Works with 10,000+ entities through virtualization, but initial capture may take time

---

## See Also

- [Monitor Window]({{ '/en/debugging/monitor' | relative_url }}) - Real-time event tracking
- [ReactiveEntitySet Guide]({{ '/en/guides/reactive-entity-sets/' | relative_url }}) - Basic usage
- [Snapshots and Persistence]({{ '/en/guides/reactive-entity-sets/persistence' | relative_url }}) - CreateSnapshot/RestoreSnapshot details
