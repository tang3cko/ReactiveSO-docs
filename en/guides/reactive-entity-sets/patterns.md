---
layout: default
title: Patterns
parent: Reactive Entity Sets
grand_parent: Guides
nav_order: 3
---

# Common Patterns

---

## Purpose

This page shows common patterns for using Reactive Entity Sets. These examples demonstrate real-world use cases you can adapt to your project.

---

## Pattern 1: Boss health bar

Track a specific boss entity's health and display it in the UI.

```csharp
public class BossHealthBar : MonoBehaviour
{
    [SerializeField] private BossEntitySetSO bossSet;
    [SerializeField] private Slider healthSlider;
    [SerializeField] private Text healthText;

    private int bossId;

    public void SetBoss(int entityId)
    {
        bossId = entityId;
        bossSet.SubscribeToEntity(bossId, OnBossStateChanged);

        if (bossSet.TryGetData(bossId, out var state))
        {
            UpdateUI(state);
        }
    }

    private void OnDisable()
    {
        if (bossId != 0)
        {
            bossSet.UnsubscribeFromEntity(bossId, OnBossStateChanged);
        }
    }

    private void OnBossStateChanged(BossState oldState, BossState newState)
    {
        UpdateUI(newState);

        if (newState.IsDead)
        {
            gameObject.SetActive(false);
        }
    }

    private void UpdateUI(BossState state)
    {
        healthSlider.value = state.HealthPercent;
        healthText.text = $"{state.Health} / {state.MaxHealth}";
    }
}
```

### Key points

- Subscribe when the boss is assigned
- Unsubscribe in OnDisable
- Update UI immediately after subscribing
- Hide UI when boss dies

---

## Pattern 2: Status effect system

Apply and track status effects across entities.

### State struct

```csharp
[Serializable]
public struct EntityStatus
{
    public bool IsPoisoned;
    public float PoisonEndTime;
    public bool IsSlowed;
    public float SlowEndTime;
}
```

### Manager class

```csharp
public class StatusEffectManager : MonoBehaviour
{
    [SerializeField] private StatusEntitySetSO statusSet;

    public void ApplyPoison(int entityId, float duration)
    {
        statusSet.UpdateData(entityId, status => {
            status.IsPoisoned = true;
            status.PoisonEndTime = Time.time + duration;
            return status;
        });
    }

    public void ApplySlow(int entityId, float duration)
    {
        statusSet.UpdateData(entityId, status => {
            status.IsSlowed = true;
            status.SlowEndTime = Time.time + duration;
            return status;
        });
    }

    private void Update()
    {
        // Check expired effects
        statusSet.ForEach((id, status) => {
            bool changed = false;

            if (status.IsPoisoned && Time.time >= status.PoisonEndTime)
            {
                status.IsPoisoned = false;
                changed = true;
            }

            if (status.IsSlowed && Time.time >= status.SlowEndTime)
            {
                status.IsSlowed = false;
                changed = true;
            }

            if (changed)
            {
                statusSet.SetData(id, status);
            }
        });
    }
}
```

### Key points

- Use UpdateData for atomic modifications
- Check expiration times in Update
- Batch multiple field changes into one SetData call

---

## Pattern 3: Save and load

Persist entity state across game sessions.

```csharp
public class SaveSystem : MonoBehaviour
{
    [SerializeField] private PlayerEntitySetSO playerSet;

    public void SaveGame()
    {
        // ForEach provides both ID and state directly - no extra lookup needed
        playerSet.ForEach((entityId, state) => {
            PlayerPrefs.SetInt($"Player_{entityId}_Health", state.Health);
            PlayerPrefs.SetInt($"Player_{entityId}_Level", state.Level);
        });
        PlayerPrefs.Save();
    }

    public void LoadEntityState(int entityId)
    {
        if (PlayerPrefs.HasKey($"Player_{entityId}_Health"))
        {
            var state = new PlayerState
            {
                Health = PlayerPrefs.GetInt($"Player_{entityId}_Health"),
                Level = PlayerPrefs.GetInt($"Player_{entityId}_Level")
            };
            playerSet.SetData(entityId, state);
        }
    }
}
```

### Key points

- Use ForEach to iterate all entities efficiently
- ForEach provides both ID and state without extra lookup
- SetData to restore loaded state

---

## Pattern 4: Damage number popup

Show damage numbers when entities take damage.

```csharp
public class DamagePopupSpawner : MonoBehaviour
{
    [SerializeField] private EnemyEntitySetSO enemySet;
    [SerializeField] private IntEventChannelSO onEnemyDataChanged;
    [SerializeField] private GameObject damagePopupPrefab;

    private Dictionary<int, int> lastHealthValues = new();

    private void OnEnable()
    {
        onEnemyDataChanged.OnEventRaised += OnEnemyChanged;

        // Initialize tracking
        enemySet.ForEach((id, state) => {
            lastHealthValues[id] = state.Health;
        });
    }

    private void OnDisable()
    {
        onEnemyDataChanged.OnEventRaised -= OnEnemyChanged;
    }

    private void OnEnemyChanged(int entityId)
    {
        if (!enemySet.TryGetData(entityId, out var state)) return;

        if (lastHealthValues.TryGetValue(entityId, out int lastHealth))
        {
            int damage = lastHealth - state.Health;
            if (damage > 0)
            {
                SpawnDamagePopup(entityId, damage);
            }
        }

        lastHealthValues[entityId] = state.Health;
    }

    private void SpawnDamagePopup(int entityId, int damage)
    {
        // Position lookup requires a separate reference (e.g., a dictionary mapping entity IDs to transforms)
        // This example assumes you have such a mapping or can find the entity another way
        var popup = Instantiate(damagePopupPrefab);
        popup.GetComponent<DamagePopup>().SetDamage(damage);
    }
}
```

### Key points

- Use set-level event to catch all changes
- Track previous values to detect damage
- Clean up tracking when entities are removed

---

## Pattern 5: Minimap with health indicators

Display entities on a minimap with health-based coloring.

```csharp
public class MinimapController : MonoBehaviour
{
    [SerializeField] private EnemyEntitySetSO enemySet;
    [SerializeField] private IntEventChannelSO onEnemyAdded;
    [SerializeField] private IntEventChannelSO onEnemyRemoved;
    [SerializeField] private GameObject iconPrefab;
    [SerializeField] private RectTransform minimapRect;

    private Dictionary<int, MinimapIcon> icons = new();

    private void OnEnable()
    {
        onEnemyAdded.OnEventRaised += CreateIcon;
        onEnemyRemoved.OnEventRaised += RemoveIcon;

        // Create icons for existing enemies
        enemySet.ForEach((id, state) => CreateIcon(id));
    }

    private void OnDisable()
    {
        onEnemyAdded.OnEventRaised -= CreateIcon;
        onEnemyRemoved.OnEventRaised -= RemoveIcon;
    }

    private void CreateIcon(int entityId)
    {
        if (icons.ContainsKey(entityId)) return;

        var iconGO = Instantiate(iconPrefab, minimapRect);
        var icon = iconGO.GetComponent<MinimapIcon>();
        icon.Initialize(entityId, enemySet);
        icons[entityId] = icon;
    }

    private void RemoveIcon(int entityId)
    {
        if (icons.TryGetValue(entityId, out var icon))
        {
            Destroy(icon.gameObject);
            icons.Remove(entityId);
        }
    }
}

public class MinimapIcon : MonoBehaviour
{
    private int entityId;
    private EnemyEntitySetSO enemySet;
    private Image iconImage;

    public void Initialize(int id, EnemyEntitySetSO set)
    {
        entityId = id;
        enemySet = set;
        iconImage = GetComponent<Image>();

        enemySet.SubscribeToEntity(entityId, OnStateChanged);
        UpdateColor();
    }

    private void OnDestroy()
    {
        enemySet.UnsubscribeFromEntity(entityId, OnStateChanged);
    }

    private void OnStateChanged(EnemyState oldState, EnemyState newState)
    {
        UpdateColor();
    }

    private void UpdateColor()
    {
        if (enemySet.TryGetData(entityId, out var state))
        {
            iconImage.color = Color.Lerp(Color.red, Color.green, state.HealthPercent);
        }
    }
}
```

### Key points

- Use OnItemAdded/OnItemRemoved for icon lifecycle
- Each icon subscribes to its own entity
- Initialize with existing entities on enable

---

## Pattern 6: Scene transition with persistent state

Maintain entity state across scene loads.

```csharp
public class SceneTransitionManager : MonoBehaviour
{
    [SerializeField] private PlayerEntitySetSO playerSet;

    // Don't clear on scene transition - data persists automatically

    public void LoadBattleScene()
    {
        // Player state survives the transition
        SceneManager.LoadScene("BattleScene");
    }
}

// In the new scene
public class BattleSceneInitializer : MonoBehaviour
{
    [SerializeField] private PlayerEntitySetSO playerSet;
    [SerializeField] private GameObject playerPrefab;

    private void Start()
    {
        // Check if player data exists from previous scene
        if (playerSet.Count > 0)
        {
            foreach (var entityId in playerSet.EntityIds)
            {
                SpawnPlayerWithExistingData(entityId);
            }
        }
    }

    private void SpawnPlayerWithExistingData(int entityId)
    {
        var player = Instantiate(playerPrefab);
        var playerComponent = player.GetComponent<Player>();
        // BindToExistingEntity is a custom method you implement to associate
        // the new GameObject with existing entity data in the set
        playerComponent.BindToExistingEntity(entityId);
    }
}
```

### Key points

- ScriptableObject data persists across scene loads
- Check for existing data when entering new scenes
- Create visual representations for existing entities

---

## Next steps

- [Best Practices](best-practices) - Learn about performance and troubleshooting
