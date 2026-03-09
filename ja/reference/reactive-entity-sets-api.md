---
layout: default
title: Reactive Entity Sets API
parent: リファレンス
nav_order: 4
---

# リアクティブエンティティセット API

## 目的

ReactiveEntitySetSOとReactiveEntityの完全なAPIリファレンスです。エンティティごとの状態管理に使うメソッド、プロパティ、イベントを記載します。

---

## 概要

Reactive Entity Setsは3つのAPIスタイルを持っています。

| APIスタイル | クラス | ユースケース |
|-----------|-------|----------|
| IDベース | `ReactiveEntitySetSO<TData>` | 直接ID操作、ネットワーク同期 |
| MonoBehaviourベース | `ReactiveEntitySetSO<TData>` | オーナー参照を使用した便利なラッパー |
| エンティティ基底クラス | `ReactiveEntity<TData>` | 自動登録パターン |

---

## ReactiveEntitySetSO\<TData\>

Sparse Setデータ構造を使用してエンティティ状態を格納するコアScriptableObjectです。

### 型制約

```csharp
public abstract class ReactiveEntitySetSO<TData> : ReactiveEntitySetSO
    where TData : unmanaged
```

TDataはunmanaged型（マネージド参照を含まない値型）でなければなりません。Job SystemおよびBurstとの互換性を確保するための制約です。

---

## プロパティ

| プロパティ | タイプ | 説明 |
|----------|------|-------------|
| Count | `int` | 登録されたエンティティ数 |
| Data | `NativeSlice<TData>` | すべての状態データへの読み取り専用アクセス（Job System互換） |
| EntityIds | `NativeSlice<int>` | すべてのエンティティIDへの読み取り専用アクセス（Job System互換） |

### イベント

| プロパティ | タイプ | 説明 |
|----------|------|-------------|
| OnItemAdded | `IntEventChannelSO` | エンティティ登録時に発火します（IDを渡します） |
| OnItemRemoved | `IntEventChannelSO` | エンティティ登録解除時に発火します（IDを渡します） |
| OnDataChanged | `IntEventChannelSO` | エンティティデータ変更時に発火します（IDを渡します） |
| OnSetChanged | `VoidEventChannelSO` | 任意の変更時に発火します |
| OnTraitAdded | `IntEventChannelSO` | Trait追加時に発火します（IDを渡します） |
| OnTraitRemoved | `IntEventChannelSO` | Trait削除時に発火します（IDを渡します） |

---

## IDベースAPI

整数IDで直接制御するためのAPIです。

### Register

```csharp
public void Register(int id, TData initialData)
```

初期状態でエンティティを登録します。

```csharp
entitySet.Register(123, new EnemyState { Health = 100 });
```

### Unregister

```csharp
public void Unregister(int id)
```

セットからエンティティを削除します。

```csharp
entitySet.Unregister(123);
```

### インデクサー

```csharp
public TData this[int id] { get; set; }
```

状態データを取得または設定します。設定時、値が異なる場合に`OnDataChanged`を発火します。

```csharp
// 取得
EnemyState state = entitySet[123];

// 設定
entitySet[123] = new EnemyState { Health = 50 };
```

### GetData

```csharp
public TData GetData(int id)
```

状態データを取得します。登録されていない場合は`KeyNotFoundException`をスローします。

### TryGetData

```csharp
public bool TryGetData(int id, out TData data)
```

例外をスローせずに状態データを取得します。

```csharp
if (entitySet.TryGetData(123, out var state))
{
    Debug.Log($"Health: {state.Health}");
}
```

### SetData

```csharp
public void SetData(int id, TData data)
```

状態データを設定します。値が異なる場合に`OnDataChanged`を発火します。

```csharp
entitySet.SetData(123, new EnemyState { Health = 50 });
```

### UpdateData

```csharp
public void UpdateData(int id, Func<TData, TData> updater)
```

変換関数で状態を更新します。`OnDataChanged`を自動的に発火します。

```csharp
entitySet.UpdateData(123, state => {
    state.Health -= damage;
    return state;
});
```

### Contains

```csharp
public bool Contains(int id)
```

IDが登録されているか確認します。

```csharp
if (entitySet.Contains(123))
{
    // エンティティが存在
}
```

### NotifyDataChanged

```csharp
public void NotifyDataChanged(int id)
```

データ変更イベントを手動で発火します。

---

## Traits API（ジェネリック）

Traitsは `ReactiveEntitySetSO<TData>` に統合されています。`TTraits` は `unmanaged, Enum` 制約があり、最初に使用した型で固定（型ロック）されます。異なる enum を混在させると `InvalidOperationException` が発生します。

Traitsは `ulong` の64bitマスクで保存されます（最大64フラグ）。

### 変更メソッド

```csharp
public void AddTraits<TTraits>(int id, TTraits traits)
```

ビットORでトレイトを追加します。`OnTraitAdded` と `OnSetChanged` が発火します。

```csharp
public void RemoveTraits<TTraits>(int id, TTraits traits)
```

ビットAND-NOTでトレイトを削除します。`OnTraitRemoved` と `OnSetChanged` が発火します。

```csharp
public void SetTraits<TTraits>(int id, TTraits traits)
```

すべてのトレイトを置き換えます。マスクが増加したか減少したかに応じて適切なトレイトイベントが発火します。

```csharp
public void ClearTraits(int id)
```

トレイトマスクを0にします。エンティティにトレイトがあった場合、`OnTraitRemoved` と `OnSetChanged` が発火します。

### クエリメソッド

```csharp
public bool HasTraits<TTraits>(int id, TTraits traits)
```

指定したフラグがすべてセットされている場合にtrueを返します。

```csharp
public bool HasAnyTrait<TTraits>(int id, TTraits traits)
```

指定したフラグのいずれかがセットされている場合にtrueを返します。

```csharp
public TTraits GetTraits<TTraits>(int id)
```

現在のトレイトを取得します。エンティティが登録されていない場合は `KeyNotFoundException` をスローします。

```csharp
public bool TryGetTraits<TTraits>(int id, out TTraits traits)
```

エンティティが登録されていない場合にfalseを返す安全なバージョンです。

### イテレーションメソッド

```csharp
public void WithTraits<TTraits>(TTraits traits, Action<int, TData> callback)
```

指定したフラグがすべてセットされているエンティティごとにコールバックを呼びます。

```csharp
public void WithAnyTraits<TTraits>(TTraits traits, Action<int, TData> callback)
```

指定したフラグのいずれかがセットされているエンティティごとにコールバックを呼びます。

```csharp
public int CountWithTraits<TTraits>(TTraits traits)
```

指定したフラグがすべてセットされているエンティティ数を返します。

```csharp
public int CountWithAnyTraits<TTraits>(TTraits traits)
```

指定したフラグのいずれかがセットされているエンティティ数を返します。

### 型ロック

`TTraits` は最初の呼び出しで型ロックされます。最初に指定した `TTraits` の enum 型がそのセットに固定されます。以降、別の enum 型で呼び出すと `InvalidOperationException` がスローされます。

### 例

```csharp
[Flags]
public enum EnemyTraits
{
    None      = 0,
    IsAggro   = 1 << 0,
    IsStunned = 1 << 1,
}

public class EnemyAISystem : MonoBehaviour
{
    [SerializeField] private EnemyEntitySetSO enemySet;

    private void Update()
    {
        // アグロ状態の敵だけを処理
        enemySet.WithTraits<EnemyTraits>(EnemyTraits.IsAggro, (id, state) =>
        {
            // アグロ敵のAIロジック
        });
    }

    public void SetAggro(int entityId, bool aggro)
    {
        if (aggro)
            enemySet.AddTraits<EnemyTraits>(entityId, EnemyTraits.IsAggro);
        else
            enemySet.RemoveTraits<EnemyTraits>(entityId, EnemyTraits.IsAggro);
    }

    public int GetAggroCount()
    {
        return enemySet.CountWithTraits<EnemyTraits>(EnemyTraits.IsAggro);
    }
}
```

### パフォーマンス

| 操作 | 計算量 |
|------|--------|
| `AddTraits` / `RemoveTraits` / `SetTraits` / `ClearTraits` | O(1) |
| `HasTraits` / `HasAnyTrait` / `GetTraits` / `TryGetTraits` | O(1) |
| `WithTraits` / `WithAnyTraits` | O(n) |
| `CountWithTraits` / `CountWithAnyTraits` | O(n) |

メモリ: エンティティごと8バイト（64bitビットマスク、最大64フラグ）。

---

## ビューAPI

ビューはセット内のエンティティをリアルタイムにフィルタリングしたサブセットです。エンティティのデータやトレイトが変わると自動的にメンバーシップを再評価し、エンティティが述語の境界を越えると `OnEnter` / `OnExit` イベントが発火します。

### ViewTrigger 列挙型

```csharp
public enum ViewTrigger
{
    None       = 0,
    DataOnly   = 1,
    TraitsOnly = 2,
    All        = 3,
}
```

| 値 | ビューが再評価するタイミング |
|----|--------------------------|
| `None` | 再評価しません。手動トリガーのビューに使います。 |
| `DataOnly` | エンティティのデータが変わったとき |
| `TraitsOnly` | エンティティのトレイトが変わったとき |
| `All` | データとトレイトの両方が変わったとき |

### CreateView（データのみ）

```csharp
public ReactiveView<TData> CreateView(
    Func<TData, bool> predicate,
    ViewTrigger triggerOn = ViewTrigger.DataOnly
)
```

データのみでエンティティをフィルタリングするビューを作成します。

```csharp
var lowHealthView = enemySet.CreateView(
    state => state.Health < state.MaxHealth * 0.3f,
    ViewTrigger.DataOnly
);
```

### CreateView（トレイト対応）

```csharp
public ReactiveView<TData> CreateView(
    Func<TData, ulong, bool> predicate,
    ViewTrigger triggerOn,
    ulong observedTraitMask
)
```

エンティティのトレイトビットマスクも受け取る述語でビューを作成します。`observedTraitMask` は、どのトレイトフラグが変わったときに再評価するかを指定します。

マスクの構築には `TraitMaskUtility.ToUInt64<TTraits>(flags)` を使います。

```csharp
ulong aggroMask = TraitMaskUtility.ToUInt64<EnemyTraits>(EnemyTraits.IsAggro);

var aggroLowHealthView = enemySet.CreateView(
    (state, traits) => (traits & aggroMask) != 0 && state.HealthPercent < 0.5f,
    ViewTrigger.All,
    aggroMask
);
```

### ReactiveView\<TData\> のメンバー

| メンバー | 説明 |
|---------|------|
| `Count` | ビューに現在含まれるエンティティ数 |
| `Contains(int id)` | O(1)のメンバーシップチェック |
| `GetEnumerator()` | メンバーIDを列挙します（foreach対応） |
| `OnEnter` | エンティティがビューに入ったとき発火します |
| `OnExit` | エンティティがビューから出たとき発火します |
| `Dispose()` | 親セットからビューを登録解除します |

### 例

```csharp
public class LowHealthSystem : MonoBehaviour
{
    [SerializeField] private EnemyEntitySetSO enemySet;

    private ReactiveView<EnemyState> lowHealthView;

    private void OnEnable()
    {
        lowHealthView = enemySet.CreateView(
            state => state.HealthPercent < 0.3f,
            ViewTrigger.DataOnly
        );

        lowHealthView.OnEnter += OnEnemyEnteredLowHealth;
        lowHealthView.OnExit += OnEnemyLeftLowHealth;
    }

    private void OnDisable()
    {
        lowHealthView.OnEnter -= OnEnemyEnteredLowHealth;
        lowHealthView.OnExit -= OnEnemyLeftLowHealth;
        lowHealthView.Dispose();
    }

    private void OnEnemyEnteredLowHealth(int entityId)
    {
        // 逃走行動の開始、警告音の再生など
    }

    private void OnEnemyLeftLowHealth(int entityId)
    {
        // 逃走行動のキャンセル
    }
}
```

{: .note }
> `OnEnter` と `OnExit` は `Clear()` や `RestoreSnapshot()` 中には発火しません。これらのケースではメンバーシップが黙って再構築されます。

### パフォーマンス

| 操作 | 計算量 |
|------|--------|
| `Contains` | O(1) |
| `GetEnumerator`（全反復） | O(n) |
| `CreateView` | O(n)（現在の全エンティティで述語を評価します） |
| エンティティ変更ごとのメンバーシップ更新 | O(v)（v = ビュー数） |

---

## MonoBehaviourベースAPI

`owner.GetInstanceID()`をエンティティIDとして使用する便利なラッパーです。

### Register

```csharp
public void Register(MonoBehaviour owner, TData initialData)
```

MonoBehaviourのインスタンスIDを使って登録します。

```csharp
entitySet.Register(this, new EnemyState { Health = 100 });
```

### Unregister

```csharp
public void Unregister(MonoBehaviour owner)
```

MonoBehaviour参照を使って登録解除します。

```csharp
entitySet.Unregister(this);
```

### インデクサー

```csharp
public TData this[MonoBehaviour owner] { get; set; }
```

オーナー参照で状態を取得または設定します。

```csharp
// 取得
EnemyState state = entitySet[this];

// 設定
entitySet[this] = new EnemyState { Health = 50 };
```

### その他のメソッド

すべてのIDベースメソッドにMonoBehaviourオーバーロードがあります。

- `GetData(MonoBehaviour owner)`
- `TryGetData(MonoBehaviour owner, out TData data)`
- `SetData(MonoBehaviour owner, TData data)`
- `UpdateData(MonoBehaviour owner, Func<TData, TData> updater)`
- `Contains(MonoBehaviour owner)`
- `NotifyDataChanged(MonoBehaviour owner)`

---

## 反復処理

### ForEach

```csharp
public void ForEach(Action<int, TData> callback)
```

すべてのエンティティを反復処理します。逆方向反復を使うため、イテレーション中の削除も安全です。

```csharp
entitySet.ForEach((id, state) => {
    if (state.Health <= 0)
    {
        entitySet.Unregister(id);  // 反復中も安全
    }
});
```

### Dataプロパティ

パフォーマンスが重要なコードでは、NativeSlice経由で直接データにアクセスできます。

```csharp
var data = entitySet.Data;
for (int i = 0; i < data.Length; i++)
{
    ProcessState(data[i]);
}
```

---

## エンティティごとの購読

特定のエンティティの変更を購読します。

### SubscribeToEntity

```csharp
public void SubscribeToEntity(int id, Action<TData, TData> callback)
```

特定のエンティティの状態変更を購読します。

```csharp
entitySet.SubscribeToEntity(123, (oldState, newState) => {
    if (oldState.Health != newState.Health)
    {
        OnHealthChanged(newState.Health);
    }
});
```

### UnsubscribeFromEntity

```csharp
public void UnsubscribeFromEntity(int id, Action<TData, TData> callback)
```

購読を削除します。

---

## ユーティリティメソッド

### Clear

```csharp
public void Clear()
```

セットからすべてのエンティティを削除します。

### CleanupDestroyed

```csharp
public void CleanupDestroyed()
```

MonoBehaviourオーナーが破棄されたエンティティを削除します。MonoBehaviourオーナーで登録されたエンティティにのみ影響します。

```csharp
// Unregisterが確実に呼ばれない場合に定期的に呼び出す
entitySet.CleanupDestroyed();
```

---

## ReactiveEntity\<TData\>

ReactiveEntitySetSOに自動登録するエンティティの基底クラスです。

### 抽象メンバー

| メンバー | タイプ | 説明 |
|--------|------|-------------|
| Set | `ReactiveEntitySetSO<TData>` | 登録先のセット |
| InitialState | `TData` | 登録時の開始状態 |

### プロパティ

| プロパティ | タイプ | 説明 |
|----------|------|-------------|
| EntityId | `int` | このエンティティの一意なID (GetInstanceID) |
| State | `TData` | セット内の状態を取得/設定 (protected) |
| IsRegistered | `bool` | 現在登録されているかどうか (protected) |

### イベント

| イベント | タイプ | 説明 |
|-------|------|-------------|
| OnStateChanged | `Action<TData, TData>` | このエンティティの状態変更時に発火 |

### 仮想メソッド

| メソッド | 説明 |
|--------|-------------|
| OnEnable | セットに登録し、変更を購読 |
| OnDisable | 購読解除して登録解除 |
| OnBeforeUnregister | 登録解除前に呼ばれます（クリーンアップ用にオーバーライド） |

---

## 実装例

### 状態構造体

```csharp
[System.Serializable]
public struct EnemyState
{
    public int Health;
    public int MaxHealth;
    public bool IsAggressive;

    public float HealthPercent => (float)Health / MaxHealth;
}
```

### セットScriptableObject

```csharp
using Tang3cko.ReactiveSO;
using UnityEngine;

[CreateAssetMenu(
    fileName = "EnemyEntitySet",
    menuName = "Game/Enemy Entity Set"
)]
public class EnemyEntitySetSO : ReactiveEntitySetSO<EnemyState>
{
    // 必要に応じてカスタムメソッドを追加
    public int CountLowHealth(float threshold)
    {
        int count = 0;
        ForEach((id, state) => {
            if (state.HealthPercent < threshold) count++;
        });
        return count;
    }
}
```

### エンティティMonoBehaviour

```csharp
using Tang3cko.ReactiveSO;
using UnityEngine;

public class Enemy : ReactiveEntity<EnemyState>
{
    [SerializeField] private EnemyEntitySetSO enemySet;
    [SerializeField] private int maxHealth = 100;

    protected override ReactiveEntitySetSO<EnemyState> Set => enemySet;

    protected override EnemyState InitialState => new EnemyState
    {
        Health = maxHealth,
        MaxHealth = maxHealth,
        IsAggressive = false
    };

    protected override void OnEnable()
    {
        base.OnEnable();
        OnStateChanged += HandleStateChanged;
    }

    protected override void OnDisable()
    {
        OnStateChanged -= HandleStateChanged;
        base.OnDisable();
    }

    public void TakeDamage(int damage)
    {
        var state = State;
        state.Health -= damage;
        State = state;

        if (state.Health <= 0)
        {
            Die();
        }
    }

    private void HandleStateChanged(EnemyState oldState, EnemyState newState)
    {
        if (oldState.Health != newState.Health)
        {
            UpdateHealthBar(newState.HealthPercent);
        }
    }

    protected override void OnBeforeUnregister()
    {
        Debug.Log($"Enemy dying with {State.Health} HP");
    }

    private void Die()
    {
        Destroy(gameObject);
    }
}
```

---

## パフォーマンス特性

| 操作 | 計算量 |
|-----------|------------|
| Register | O(1) 償却 |
| Unregister | O(1) |
| GetData / SetData | O(1) |
| Contains | O(1) |
| ForEach | O(n) |
| CleanupDestroyed | O(n) |

Sparse SetはO(1)アクセスを提供しつつ、反復処理に最適なキャッシュフレンドリーな連続ストレージを維持します。

---

## スレッドセーフティ

ReactiveEntitySetSOは**スレッドセーフではありません**。すべての操作はUnityメインスレッドで実行してください。Jobs、asyncコンテキスト、バックグラウンドスレッドからアクセスすると未定義動作になります。

---

## 参考資料

- [リアクティブエンティティセットガイド]({{ '/ja/guides/reactive-entity-sets' | relative_url }}) - リアクティブエンティティセットの使い方
- [トレイトガイド]({{ '/ja/guides/reactive-entity-sets/traits' | relative_url }}) - トレイト enum の定義と使い方
- [ビューガイド]({{ '/ja/guides/reactive-entity-sets/views' | relative_url }}) - ビューの作成と管理
- [ランタイムセットリファレンス](runtime-set-types) - シンプルなオブジェクト追跡
- [イベントタイプリファレンス](event-types) - リアクティブエンティティセットが使用するイベントチャンネル
