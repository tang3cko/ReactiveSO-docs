---
layout: default
title: Variable monitor
parent: Debugging
nav_order: 3
---

# Variable monitor

{: .note }
> Available since v1.1.0.

## Purpose

This guide explains how to use the Variable Monitor Window to track variable values in real-time. You will learn to inspect current values, filter by type, and quickly locate variable assets during debugging.

---

## Opening the window

Navigate to **Window > Reactive SO > Variable Monitor**.

<!-- TODO: Add screenshot of Variable Monitor Window showing list of variables with their current values -->

---

## Window columns

| Column | Description |
|--------|-------------|
| Name | Name of the variable asset |
| Type | Variable type (Int, Float, Bool, etc.) |
| Value | Current value during Play Mode |
| Path | Asset path in the Project |

---

## Features

### Type filtering

Filter variables by their type using the type dropdown. Available filters include Int, Float, Bool, String, Vector2, Vector3, Color, Quaternion, and more.

### Search

Type in the search box to filter variables by name. Partial matches work, so typing "Health" shows both `PlayerHealth` and `EnemyHealth`.

### Ping asset

Click any variable row to ping (highlight) the asset in the Project window. This helps you quickly locate and select the asset for inspection.

### Formatted display

Special value types display in readable formats.

| Type | Display Format |
|------|----------------|
| Vector2 | (x, y) |
| Vector3 | (x, y, z) |
| Quaternion | (x, y, z, w) |
| Color | RGBA with color preview |
| Bool | True/False |

---

## Use cases

### Inspecting game state

During Play Mode, open the Variable Monitor to see all variable values at a glance. This is faster than selecting each asset individually in the Inspector.

### Finding unexpected values

When game behavior seems wrong, check the Variable Monitor for values that don't match expectations. For example, if player damage seems off, verify that `PlayerDamageMultiplier` shows the expected value.

### Verifying initialization

After scene load, confirm that variables have their expected initial values. Variables with incorrect initial values often indicate missing `ResetToInitial()` calls.

### Debugging cross-scene state

Since variables persist across scenes, use the Variable Monitor to verify that values carry over correctly during scene transitions.

---

## Real-time updates

Values update automatically during Play Mode. The window refreshes when any variable's value changes, so you see current state without manual refresh.

Outside Play Mode, values reflect the serialized asset state (typically the Initial Value).

---

## Tips

- Keep the window docked next to the Game view during playtesting
- Use type filtering to focus on specific variable categories
- Click a row to ping the asset, then check the Inspector for GPU Sync settings
- Combine with Event Monitor to correlate value changes with events

---

## References

- [Variables Guide]({{ '/en/guides/variables' | relative_url }}) - How to use variables
- [Debugging Overview]({{ '/en/debugging/' | relative_url }}) - All debugging tools
- [Event Monitor](event-monitor) - Real-time event tracking
