---
layout: default
title: Runtime Set Monitor
parent: Debugging
nav_order: 5
---

# Runtime set monitor

{: .note }
> Available since v1.1.0.

## Purpose

This guide explains how to use the Runtime Set Monitor Window to track runtime set contents in real-time. You will learn to inspect which objects are currently in each set and monitor collection changes during gameplay.

---

## Opening the window

Navigate to **Window > Reactive SO > Runtime Set Monitor**.

<!-- TODO: Add screenshot of Runtime Set Monitor Window showing list of runtime sets with their item counts -->

---

## Window columns

| Column | Description |
|--------|-------------|
| Runtime Set | Name of the runtime set asset |
| Type | Element type (GameObject, Transform, etc.) |
| Count | Number of items currently in the set |
| Status | Active (green) when count > 0, Empty (gray) when count = 0 |
| -> | Button to ping and select asset in Project window |

---

## Features

### Type filtering

Filter runtime sets by their element type using the type dropdown. This helps focus on specific categories like GameObjects or Transforms.

### Search

Type in the search box to filter sets by name. Useful when you have many runtime sets and need to find a specific one quickly.

### Ping asset

Click the -> button or double-click any row to ping the asset in the Project window. The asset is also selected, so you can immediately view its contents in the Inspector.

### Real-time count

The Count and Status columns refresh automatically every 200ms during Play Mode. The [LIVE] indicator appears in the footer when auto-refresh is active.

### Status indicator

The Status column provides quick visual feedback about set state.

- **Active** (green text) - Contains one or more items
- **Empty** (gray text) - Contains no items

### Refresh button

Click the Refresh button in the toolbar to manually reload all runtime sets from the project. Use this after creating new runtime set assets to update the list without reopening the window.

---

## Use cases

### Verifying object registration

Check that spawned enemies are correctly registering themselves. If you spawn 5 enemies but the count shows 3, some enemies may be failing to register.

### Debugging object cleanup

After destroying objects, verify they unregister from sets. A count that doesn't decrease after `Destroy()` indicates missing `OnDisable` unregistration.

### Monitoring population limits

For systems with spawn limits, watch the count to verify limits are enforced. If `ActiveEnemies` should never exceed 10 but shows 12, the spawn limiter has a bug.

### Scene transition verification

After scene transitions, confirm that sets are cleared or preserved as expected. Persistent sets should maintain their counts while scene-specific sets should reset.

---

## Viewing set contents

The Runtime Set Monitor shows counts, not contents. To see the actual items in a set, select the runtime set asset and check the Inspector during Play Mode.

The Inspector shows a Runtime Items list with each object in the set. Click any item to ping it in the Hierarchy.

---

## Common issues

### Count stays at zero

If a runtime set's count never increases during gameplay, check that objects call `Add()` in their `OnEnable` method and that the correct asset is assigned.

### Count keeps growing

If count increases but never decreases, objects are not unregistering. Ensure `Remove()` is called in `OnDisable`, not `OnDestroy`.

### Null entries appear

If the Inspector shows null entries, objects were destroyed without proper unregistration. Call `CleanupDestroyed()` or fix the registration pattern.

---

## Tips

- Monitor counts while playtesting to catch registration bugs early
- Compare expected vs actual counts after spawning objects
- Use alongside Event Monitor to correlate registration events
- Check set contents in Inspector for detailed object information

---

## References

- [Runtime Sets Guide]({{ '/en/guides/runtime-sets' | relative_url }}) - How to use runtime sets
- [Debugging Overview]({{ '/en/debugging/' | relative_url }}) - All debugging tools
- [Variable Monitor](variable-monitor) - Real-time variable tracking
