---
layout: default
title: Event types
parent: Reference
nav_order: 1
---

# Event types

## Purpose

This reference documents all 12 built-in event channel types. You will find the use cases, code examples, and creation instructions for each type.

---

## Type overview

| Type | Parameter | Example Use Cases |
|------|-----------|-------------------|
| Void | None | Game start, player death, level complete |
| Int | `int` | Score changed, level up, enemy count |
| Long | `long` | Timestamps, large numbers |
| Float | `float` | Health percent, timer, volume |
| Double | `double` | High-precision values |
| Bool | `bool` | Game paused, feature enabled |
| String | `string` | Dialogue, notifications, names |
| Vector2 | `Vector2` | Input axis, touch position |
| Vector3 | `Vector3` | Spawn position, target location |
| Quaternion | `Quaternion` | Camera rotation, orientation |
| Color | `Color` | Theme color, UI tint |
| GameObject | `GameObject` | Enemy spawned, target changed |

---

## Choosing the right type

Use this decision guide:

1. **No data needed?** Use Void
2. **Whole numbers?**
   - Standard range (-2B to +2B): Use Int
   - Large numbers or timestamps: Use Long
3. **Decimal numbers?**
   - Standard precision: Use Float
   - High precision: Use Double
4. **True/False?** Use Bool
5. **Text?** Use String
6. **2D position/direction?** Use Vector2
7. **3D position/direction?** Use Vector3
8. **3D rotation?** Use Quaternion
9. **Color value?** Use Color
10. **Object reference?** Use GameObject
11. **Complex data?** Create a custom type

---

## Void

Events with no parameters. Use when the event itself carries all necessary information.

### Create

```
Create > Reactive SO > Channels > Void Event
```

### Usage

```csharp
[SerializeField] private VoidEventChannelSO onPlayerDeath;

// Publisher
public void Die()
{
    onPlayerDeath?.RaiseEvent();
}

// Subscriber
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
    ShowGameOverScreen();
}
```

### Common use cases

- Player death / respawn
- Game pause / resume
- Level complete / failed
- Menu open / close
- Achievement unlocked
- Save completed

---

## Int

Events carrying integer values for counts, scores, and indices.

### Create

```
Create > Reactive SO > Channels > Int Event
```

### Usage

```csharp
[SerializeField] private IntEventChannelSO onScoreChanged;

// Publisher
public void AddScore(int points)
{
    currentScore += points;
    onScoreChanged?.RaiseEvent(currentScore);
}

// Subscriber
private void OnEnable()
{
    onScoreChanged.OnEventRaised += UpdateScoreDisplay;
}

private void OnDisable()
{
    onScoreChanged.OnEventRaised -= UpdateScoreDisplay;
}

private void UpdateScoreDisplay(int newScore)
{
    scoreText.text = $"Score: {newScore}";
}
```

### Common use cases

- Score updates
- Currency changes (gold, gems)
- Inventory counts
- Level / experience
- Wave / round numbers
- Enemy kill counts
- Entity IDs

---

## Long

Events carrying 64-bit integer values for large numbers and timestamps.

### Create

```
Create > Reactive SO > Channels > Long Event
```

### Usage

```csharp
[SerializeField] private LongEventChannelSO onTimestamp;

// Publisher
public void UpdateTimestamp()
{
    long timestamp = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();
    onTimestamp?.RaiseEvent(timestamp);
}

// Subscriber
private void OnEnable()
{
    onTimestamp.OnEventRaised += HandleTimestamp;
}

private void OnDisable()
{
    onTimestamp.OnEventRaised -= HandleTimestamp;
}

private void HandleTimestamp(long timestamp)
{
    Debug.Log($"Timestamp: {timestamp}");
}
```

### Common use cases

- Unix timestamps
- Large score values (> 2 billion)
- Unique ID generation
- Frame counters in long sessions
- File sizes
- Database keys

---

## Float

Events carrying floating-point values for percentages, health, and time.

### Create

```
Create > Reactive SO > Channels > Float Event
```

### Usage

```csharp
[SerializeField] private FloatEventChannelSO onHealthChanged;

// Publisher
public void TakeDamage(float damage)
{
    currentHealth -= damage;
    float healthPercent = currentHealth / maxHealth;
    onHealthChanged?.RaiseEvent(healthPercent);
}

// Subscriber
private void OnEnable()
{
    onHealthChanged.OnEventRaised += UpdateHealthBar;
}

private void OnDisable()
{
    onHealthChanged.OnEventRaised -= UpdateHealthBar;
}

private void UpdateHealthBar(float healthPercent)
{
    healthBar.fillAmount = healthPercent;
}
```

### Common use cases

- Health / stamina / mana (normalized 0-1)
- Progress bars (loading, crafting)
- Timer countdowns
- Volume / pitch settings
- Speed multipliers
- Temperature values

---

## Double

Events carrying double-precision values for high-precision calculations.

### Create

```
Create > Reactive SO > Channels > Double Event
```

### Usage

```csharp
[SerializeField] private DoubleEventChannelSO onPreciseValue;

// Publisher
public void CalculateDistance()
{
    double distance = Vector3.Distance(pointA, pointB);
    onPreciseValue?.RaiseEvent(distance);
}

// Subscriber
private void OnEnable()
{
    onPreciseValue.OnEventRaised += HandleValue;
}

private void OnDisable()
{
    onPreciseValue.OnEventRaised -= HandleValue;
}

private void HandleValue(double value)
{
    Debug.Log($"Precise: {value:F10}");
}
```

### Common use cases

- Scientific calculations
- Financial values (high precision currency)
- Large coordinate systems
- Physics simulations
- Statistical calculations

---

## Bool

Events carrying boolean values for state toggles and flags.

### Create

```
Create > Reactive SO > Channels > Bool Event
```

### Usage

```csharp
[SerializeField] private BoolEventChannelSO onGamePaused;

// Publisher
public void TogglePause()
{
    isPaused = !isPaused;
    onGamePaused?.RaiseEvent(isPaused);
}

// Subscriber
private void OnEnable()
{
    onGamePaused.OnEventRaised += HandlePause;
}

private void OnDisable()
{
    onGamePaused.OnEventRaised -= HandlePause;
}

private void HandlePause(bool isPaused)
{
    Time.timeScale = isPaused ? 0f : 1f;
}
```

### Common use cases

- Pause / resume state
- Mute / unmute audio
- Lock / unlock doors
- Enable / disable features
- Day / night cycle
- Power-up active state

---

## String

Events carrying text data for messages, names, and identifiers.

### Create

```
Create > Reactive SO > Channels > String Event
```

### Usage

```csharp
[SerializeField] private StringEventChannelSO onDialogue;

// Publisher
public void StartDialogue(string dialogueKey)
{
    onDialogue?.RaiseEvent(dialogueKey);
}

// Subscriber
private void OnEnable()
{
    onDialogue.OnEventRaised += ShowDialogue;
}

private void OnDisable()
{
    onDialogue.OnEventRaised -= ShowDialogue;
}

private void ShowDialogue(string dialogueKey)
{
    string text = dialogueDatabase.GetText(dialogueKey);
    dialogueBox.Display(text);
}
```

### Common use cases

- Dialogue / cutscene triggers
- Quest names / descriptions
- Item names
- Player names
- Status messages
- Error messages
- Scene names for loading

---

## Vector2

Events carrying 2D vector data for positions, input, and screen coordinates.

### Create

```
Create > Reactive SO > Channels > Vector2 Event
```

### Usage

```csharp
[SerializeField] private Vector2EventChannelSO onInputAxis;

// Publisher
public void OnMove(InputAction.CallbackContext context)
{
    Vector2 input = context.ReadValue<Vector2>();
    onInputAxis?.RaiseEvent(input);
}

// Subscriber
private void OnEnable()
{
    onInputAxis.OnEventRaised += HandleMovement;
}

private void OnDisable()
{
    onInputAxis.OnEventRaised -= HandleMovement;
}

private void HandleMovement(Vector2 input)
{
    Vector3 movement = new Vector3(input.x, 0, input.y);
    transform.Translate(movement * speed * Time.deltaTime);
}
```

### Common use cases

- 2D movement input (WASD, joystick)
- 2D world positions
- UI element positions
- Touch / mouse screen positions
- Minimap coordinates
- 2D velocities

---

## Vector3

Events carrying 3D vector data for positions, directions, and velocities.

### Create

```
Create > Reactive SO > Channels > Vector3 Event
```

### Usage

```csharp
[SerializeField] private Vector3EventChannelSO onSpawnPosition;

// Publisher
public void RequestSpawn()
{
    Vector3 position = GetRandomSpawnPoint();
    onSpawnPosition?.RaiseEvent(position);
}

// Subscriber
private void OnEnable()
{
    onSpawnPosition.OnEventRaised += SpawnEnemy;
}

private void OnDisable()
{
    onSpawnPosition.OnEventRaised -= SpawnEnemy;
}

private void SpawnEnemy(Vector3 position)
{
    Instantiate(enemyPrefab, position, Quaternion.identity);
}
```

### Common use cases

- Spawn positions
- Target / waypoint positions
- Look directions
- Velocities / forces
- Impact points (bullets, explosions)
- Camera positions
- Teleport destinations

---

## Quaternion

Events carrying rotation values for 3D orientations.

### Create

```
Create > Reactive SO > Channels > Quaternion Event
```

### Usage

```csharp
[SerializeField] private QuaternionEventChannelSO onCameraRotation;

// Publisher
public void RotateCamera()
{
    Quaternion rotation = cameraTransform.rotation;
    onCameraRotation?.RaiseEvent(rotation);
}

// Subscriber
private void OnEnable()
{
    onCameraRotation.OnEventRaised += HandleRotation;
}

private void OnDisable()
{
    onCameraRotation.OnEventRaised -= HandleRotation;
}

private void HandleRotation(Quaternion rotation)
{
    targetObject.rotation = rotation;
}
```

### Common use cases

- Camera rotation sync
- Object orientation changes
- Character look rotation
- Procedural rotation
- Network rotation sync

---

## Color

Events carrying color values for UI theming and visual effects.

### Create

```
Create > Reactive SO > Channels > Color Event
```

### Usage

```csharp
[SerializeField] private ColorEventChannelSO onThemeColor;

// Publisher
public void ChangeTheme(Color newColor)
{
    onThemeColor?.RaiseEvent(newColor);
}

// Subscriber
private void OnEnable()
{
    onThemeColor.OnEventRaised += HandleColorChange;
}

private void OnDisable()
{
    onThemeColor.OnEventRaised -= HandleColorChange;
}

private void HandleColorChange(Color color)
{
    uiPanel.color = color;
}
```

### Common use cases

- UI theme changes
- Health bar color indicators
- Material tint sync
- Visual effect colors
- Team color assignment
- Lighting color changes

---

## GameObject

Events carrying object references for spawned entities and targets.

### Create

```
Create > Reactive SO > Channels > GameObject Event
```

### Usage

```csharp
[SerializeField] private GameObjectEventChannelSO onEnemySpawned;

// Publisher
public void SpawnEnemy()
{
    GameObject enemy = Instantiate(enemyPrefab);
    onEnemySpawned?.RaiseEvent(enemy);
}

// Subscriber
private void OnEnable()
{
    onEnemySpawned.OnEventRaised += TrackEnemy;
}

private void OnDisable()
{
    onEnemySpawned.OnEventRaised -= TrackEnemy;
}

private void TrackEnemy(GameObject enemy)
{
    if (enemy != null)
    {
        activeEnemies.Add(enemy);
    }
}
```

### Common use cases

- Spawned object notifications
- Target acquisition / loss
- Selected object changes
- Picked up items
- Triggered objects
- Player / enemy references
- Camera targets

### Notes

- GameObject channels can pass `null` to indicate "no object"
- Always null-check the received GameObject before using it
- The object may be destroyed between raise and receive

---

## Creating custom types

If the built-in types do not fit your needs, create a custom event type:

```csharp
using Tang3cko.ReactiveSO;
using UnityEngine;

[CreateAssetMenu(
    fileName = "WeaponEvent",
    menuName = "Reactive SO/Channels/Weapon Event"
)]
public class WeaponEventChannelSO : EventChannelSO<WeaponData>
{
    // Inherits RaiseEvent(WeaponData value) automatically
}

[System.Serializable]
public struct WeaponData
{
    public string Name;
    public int Damage;
    public float Range;
}
```

Custom types work with Event Monitor and Subscribers List but do not have Manual Trigger support.

---

## References

- [Event Channels Guide]({{ '/en/guides/event-channels' | relative_url }}) - How to use event channels
- [Variables Guide]({{ '/en/guides/variables' | relative_url }}) - For reactive state
- [Debugging Overview]({{ '/en/debugging/' | relative_url }}) - Debug event flow
