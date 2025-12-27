---
layout: default
title: パターン
parent: Reactive Entity Sets
grand_parent: ガイド
nav_order: 3
---

# 共通パターン

---

## 目的

このページでは、Reactive Entity Setsを使用する際の一般的なパターンを紹介します。これらの例は、プロジェクトに適応できる実世界のユースケースを示しています。

---

## パターン1: ボスヘルスバー

特定のボスエンティティの体力を追跡してUIに表示します。

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

### ポイント

- ボスが割り当てられたときにサブスクライブ
- OnDisableでサブスクライブ解除
- サブスクライブ直後にUIを更新
- ボスが死亡したらUIを非表示

---

## パターン2: ステータスエフェクトシステム

エンティティ全体にステータスエフェクトを適用して追跡します。

### State構造体

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

### マネージャークラス

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
        // 期限切れのエフェクトを確認
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

### ポイント

- アトミックな変更にはUpdateDataを使用
- Updateで期限切れ時間を確認
- 複数フィールドの変更を1つのSetDataにまとめる

---

## パターン3: セーブとロード

ゲームセッション間でエンティティ状態を永続化します。

```csharp
public class SaveSystem : MonoBehaviour
{
    [SerializeField] private PlayerEntitySetSO playerSet;

    public void SaveGame()
    {
        foreach (var entityId in playerSet.EntityIds)
        {
            if (playerSet.TryGetData(entityId, out var state))
            {
                PlayerPrefs.SetInt($"Player_{entityId}_Health", state.Health);
                PlayerPrefs.SetInt($"Player_{entityId}_Level", state.Level);
            }
        }
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

### ポイント

- EntityIdsを反復してすべての登録エンティティを取得
- 安全なアクセスにはTryGetDataを使用
- ロードした状態を復元するにはSetDataを使用

---

## パターン4: ダメージ数字ポップアップ

エンティティがダメージを受けたときにダメージ数字を表示します。

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

        // 追跡を初期化
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
        // エンティティの位置を見つけてポップアップをスポーン
        var popup = Instantiate(damagePopupPrefab);
        popup.GetComponent<DamagePopup>().SetDamage(damage);
    }
}
```

### ポイント

- すべての変更をキャッチするためにセットレベルのイベントを使用
- ダメージを検出するために前回の値を追跡
- エンティティが削除されたときに追跡をクリーンアップ

---

## パターン5: 体力インジケータ付きミニマップ

体力に基づく色付けでミニマップにエンティティを表示します。

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

        // 既存の敵のアイコンを作成
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

### ポイント

- アイコンのライフサイクルにはOnItemAdded/OnItemRemovedを使用
- 各アイコンは自身のエンティティをサブスクライブ
- 有効化時に既存のエンティティで初期化

---

## パターン6: 永続状態を持つシーン遷移

シーンロード間でエンティティ状態を維持します。

```csharp
public class SceneTransitionManager : MonoBehaviour
{
    [SerializeField] private PlayerEntitySetSO playerSet;

    // シーン遷移時にクリアしない - データは自動的に永続化

    public void LoadBattleScene()
    {
        // プレイヤー状態は遷移を生き残る
        SceneManager.LoadScene("BattleScene");
    }
}

// 新しいシーンで
public class BattleSceneInitializer : MonoBehaviour
{
    [SerializeField] private PlayerEntitySetSO playerSet;
    [SerializeField] private GameObject playerPrefab;

    private void Start()
    {
        // 前のシーンからプレイヤーデータが存在するか確認
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
        playerComponent.BindToExistingEntity(entityId);
    }
}
```

### ポイント

- ScriptableObjectデータはシーンロード間で永続化
- 新しいシーンに入るときに既存のデータを確認
- 既存のエンティティ用に視覚的表現を作成

---

## 次のステップ

- [ベストプラクティス](best-practices) - パフォーマンスとトラブルシューティングについて学ぶ
