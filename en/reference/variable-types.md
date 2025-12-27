---
layout: default
title: Variable types
parent: Reference
nav_order: 2
---

# Variable types

{: .note }
> Variables are available since v1.1.0. GPU Sync was added in v2.0.0.

## Purpose

This reference documents all 11 built-in variable types. You will find the use cases, GPU Sync compatibility, and code examples for each type.

---

## Type overview

| Type | GPU Sync | Example Use Cases |
|------|----------|-------------------|
| Int | ✅ | Score, health, currency, level |
| Long | ❌ | Timestamps, large numbers |
| Float | ✅ | Timer, volume, progress |
| Double | ❌ | High-precision values |
| Bool | ✅ | Game paused, feature enabled |
| String | ❌ | Player name, status message |
| Vector2 | ✅ | 2D position, input axis |
| Vector3 | ✅ | 3D position, velocity |
| Quaternion | ✅ | Rotation, orientation |
| Color | ✅ | Theme color, material tint |
| GameObject | ❌ | Current target, selected object |

---

## GPU Sync support

GPU Sync allows variable values to be automatically synchronized to shader global properties. This enables shaders, VFX Graph, and Compute Shaders to react to gameplay state without additional bridging code.

| Variable Type | Shader Method | HLSL Type |
|---------------|---------------|-----------|
| Int | `SetGlobalInteger` | `int` |
| Float | `SetGlobalFloat` | `float` |
| Bool | `SetGlobalInteger` | `int` (0 or 1) |
| Vector2 | `SetGlobalVector` | `float4` (xy used) |
| Vector3 | `SetGlobalVector` | `float4` (xyz used) |
| Quaternion | `SetGlobalVector` | `float4` (xyzw) |
| Color | `SetGlobalColor` | `float4` |

String, Long, Double, and GameObject do not support GPU Sync due to their data types.

---

## Common API

All variable types share this API.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| Value | `T` | Current value (get/set) |
| InitialValue | `T` | Value set in Inspector |
| OnValueChanged | `EventChannelSO<T>` | Event channel to notify changes |

### Methods

| Method | Description |
|--------|-------------|
| `ResetToInitial()` | Restore Value to InitialValue |
| `NotifyValueChanged()` | Force event notification without value change |

### Events

Setting `Value` automatically raises `OnValueChanged` if the new value differs from the current value. This uses `EqualityComparer<T>` for comparison.

---

## Int

Integer variables for whole numbers within standard range.

### Create

```text
Create > Reactive SO > Variables > Int Variable
```

### Usage

```csharp
[SerializeField] private IntVariableSO playerScore;

// Write
playerScore.Value += 100;

// Read
int score = playerScore.Value;

// Listen to changes (via linked event channel)
onScoreChanged.OnEventRaised += UpdateScoreUI;
```

### GPU Sync

```hlsl
// In shader (after enabling GPU Sync)
int score = _PlayerScore;
```

### Common use cases

- Player score
- Currency (gold, gems)
- Health / stamina (integer-based)
- Level / experience
- Wave / round numbers
- Inventory quantities

---

## Long

64-bit integer variables for large numbers and timestamps.

### Create

```text
Create > Reactive SO > Variables > Long Variable
```

### Usage

```csharp
[SerializeField] private LongVariableSO lastSaveTime;

// Write
lastSaveTime.Value = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();

// Read
long timestamp = lastSaveTime.Value;
```

### Common use cases

- Unix timestamps
- Large score values (> 2 billion)
- Unique identifiers
- Frame counters in long sessions
- Database keys

---

## Float

Floating-point variables for decimal values and percentages.

### Create

```text
Create > Reactive SO > Variables > Float Variable
```

### Usage

```csharp
[SerializeField] private FloatVariableSO masterVolume;

// Write
masterVolume.Value = 0.75f;

// Read
audioSource.volume = masterVolume.Value;
```

### GPU Sync

```hlsl
// In shader
float volume = _MasterVolume;
float healthFactor = saturate(_PlayerHealthPercent);
```

### Common use cases

- Volume settings
- Health / stamina / mana (normalized 0-1)
- Progress values (loading, crafting)
- Timer countdowns
- Speed multipliers
- Spawn rates

---

## Double

Double-precision variables for high-accuracy calculations.

### Create

```text
Create > Reactive SO > Variables > Double Variable
```

### Usage

```csharp
[SerializeField] private DoubleVariableSO preciseDistance;

// Write
preciseDistance.Value = Vector3.Distance(a, b);

// Read
double distance = preciseDistance.Value;
```

### Common use cases

- Scientific calculations
- Financial values (high precision)
- Large coordinate systems
- Statistical calculations

---

## Bool

Boolean variables for true/false states.

### Create

```text
Create > Reactive SO > Variables > Bool Variable
```

### Usage

```csharp
[SerializeField] private BoolVariableSO isPaused;

// Write
isPaused.Value = true;

// Read
if (isPaused.Value) Time.timeScale = 0;
```

### GPU Sync

```hlsl
// In shader (0 = false, 1 = true)
int paused = _IsPaused;
if (paused) { /* apply pause effect */ }
```

### Common use cases

- Game paused state
- Feature toggles
- Mute settings
- Player alive state
- Debug mode enabled
- Tutorial completed

---

## String

Text variables for names, messages, and identifiers.

### Create

```text
Create > Reactive SO > Variables > String Variable
```

### Usage

```csharp
[SerializeField] private StringVariableSO playerName;

// Write
playerName.Value = "Hero";

// Read
nameLabel.text = playerName.Value;
```

### Common use cases

- Player name
- Current level name
- Status messages
- Localization keys
- Save slot names

---

## Vector2

2D vector variables for positions and input.

### Create

```text
Create > Reactive SO > Variables > Vector2 Variable
```

### Usage

```csharp
[SerializeField] private Vector2VariableSO inputAxis;

// Write
inputAxis.Value = new Vector2(Input.GetAxis("Horizontal"), Input.GetAxis("Vertical"));

// Read
Vector2 input = inputAxis.Value;
```

### GPU Sync

```hlsl
// In shader
float4 input = _InputAxis;  // xy contains the Vector2 values
float2 axis = input.xy;
```

### Common use cases

- Input axis values
- 2D positions
- Screen positions
- Minimap coordinates
- Joystick values

---

## Vector3

3D vector variables for positions and directions.

### Create

```text
Create > Reactive SO > Variables > Vector3 Variable
```

### Usage

```csharp
[SerializeField] private Vector3VariableSO playerPosition;

// Write
playerPosition.Value = transform.position;

// Read
Vector3 pos = playerPosition.Value;
```

### GPU Sync

```hlsl
// In shader
float4 pos = _PlayerPosition;  // xyz contains the Vector3 values
float3 playerPos = pos.xyz;
float distance = length(worldPos - playerPos);
```

### Common use cases

- Player position (for shaders)
- Target positions
- Spawn points
- Wind direction
- Light direction

---

## Quaternion

Rotation variables for 3D orientations.

### Create

```text
Create > Reactive SO > Variables > Quaternion Variable
```

### Usage

```csharp
[SerializeField] private QuaternionVariableSO cameraRotation;

// Write
cameraRotation.Value = Camera.main.transform.rotation;

// Read
Quaternion rot = cameraRotation.Value;
```

### GPU Sync

```hlsl
// In shader
float4 rot = _CameraRotation;  // xyzw contains quaternion components
```

### Common use cases

- Camera orientation
- Object rotations
- Procedural rotation

---

## Color

Color variables for theming and visual effects.

### Create

```text
Create > Reactive SO > Variables > Color Variable
```

### Usage

```csharp
[SerializeField] private ColorVariableSO themeColor;

// Write
themeColor.Value = Color.red;

// Read
uiImage.color = themeColor.Value;
```

### GPU Sync

```hlsl
// In shader
float4 theme = _ThemeColor;  // RGBA values
float3 rgb = theme.rgb;
```

### Common use cases

- UI theme color
- Danger indicator color
- Health bar tint
- Environment fog color
- Lighting color

---

## GameObject

Object reference variables for tracking targets.

### Create

```text
Create > Reactive SO > Variables > GameObject Variable
```

### Usage

```csharp
[SerializeField] private GameObjectVariableSO currentTarget;

// Write
currentTarget.Value = hitEnemy;

// Read
if (currentTarget.Value != null)
{
    attackTarget = currentTarget.Value;
}
```

### Notes

- Can hold `null` to indicate "no object"
- Reference may become invalid if object is destroyed
- Always null-check before use
- No GPU Sync support

### Common use cases

- Current target reference
- Selected object
- Interacting object
- Closest enemy
- Active weapon

---

## Creating custom types

Create a custom variable type by extending `VariableSO<T>`.

```csharp
using Tang3cko.ReactiveSO;
using UnityEngine;

[CreateAssetMenu(
    fileName = "WeaponVariable",
    menuName = "Reactive SO/Variables/Weapon Variable"
)]
public class WeaponVariableSO : VariableSO<WeaponData>
{
    // Inherits Value, InitialValue, ResetToInitial() automatically
}

[System.Serializable]
public struct WeaponData
{
    public string Name;
    public int Damage;
    public float Range;
}
```

Create a matching event channel for change notifications.

```csharp
[CreateAssetMenu(
    fileName = "WeaponEvent",
    menuName = "Reactive SO/Channels/Weapon Event"
)]
public class WeaponEventChannelSO : EventChannelSO<WeaponData>
{
}
```

Custom types do not support GPU Sync unless you override `SyncValueToGPU()`.

---

## References

- [Variables Guide]({{ '/en/guides/variables' | relative_url }}) - How to use variables
- [Event Types Reference](event-types) - All event channel types
- [Debugging Overview]({{ '/en/debugging/' | relative_url }}) - Debug variable values
