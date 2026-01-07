---
layout: default
title: Reactive Entity Sets API
parent: リファレンス
nav_order: 4
---

# リアクティブエンティティセット API

## 目的

このリファレンスでは、ReactiveEntitySetSOとReactiveEntityの完全なAPIを解説します。エンティティごとの状態管理のためのメソッド、プロパティ、イベントを確認できます。

---

## 概要

Reactive Entity Setsは3つのAPIスタイルを提供します。

| APIスタイル | クラス | ユースケース |
|-----------|-------|----------|
| IDベース | `ReactiveEntitySetSO<TData>` | 直接ID操作、ネットワーク同期 |
| MonoBehaviourベース | `ReactiveEntitySetSO<TData>` | オーナー参照を使用した便利なラッパー |
| エンティティ基底クラス | `ReactiveEntity<TData>` | 自動登録パターン |

---

## ReactiveEntitySetSO\<TData\>

Sparse Setデータ構造を使用してエンティティ状態を格納するコアScriptableObject。

### 型制約

```csharp
public abstract class ReactiveEntitySetSO<TData> : ReactiveEntitySetSO
    where TData : unmanaged
```

TDataはunmanaged型（マネージド参照を含まない値型）である必要があります。これによりJob SystemとBurstとの互換性が有効になります。

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
| OnItemAdded | `IntEventChannelSO` | エンティティ登録時に発火（IDを渡す） |
| OnItemRemoved | `IntEventChannelSO` | エンティティ登録解除時に発火（IDを渡す） |
| OnDataChanged | `IntEventChannelSO` | エンティティデータ変更時に発火（IDを渡す） |
| OnSetChanged | `VoidEventChannelSO` | 任意の変更時に発火 |

---

## IDベースAPI

整数IDで直接制御するためのAPI。

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

状態データを取得または設定します。設定時、値が異なる場合にOnDataChangedをトリガーします。

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

例外をスローせずに状態データの取得を試みます。

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

状態データを設定します。値が異なる場合にOnDataChangedを発火します。

```csharp
entitySet.SetData(123, new EnemyState { Health = 50 });
```

### UpdateData

```csharp
public void UpdateData(int id, Func<TData, TData> updater)
```

変換関数を使用して状態を更新します。自動的にOnDataChangedをトリガーします。

```csharp
entitySet.UpdateData(123, state => {
    state.Health -= damage;
    return state;
});
```

### GetDataRef

```csharp
public ref TData GetDataRef(int id)
```

直接変更のための状態データへの参照を取得します。頻繁な更新に対してより効率的ですが、手動での通知が必要です。

```csharp
ref var state = ref entitySet.GetDataRef(123);
state.Health -= damage;
entitySet.NotifyDataChanged(123);  // 手動で呼び出す必要あり
```

### Contains

```csharp
public bool Contains(int id)
```

IDが登録されているかを確認します。

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

手動でデータ変更イベントをトリガーします。GetDataRefでの変更後に使用します。

---

## MonoBehaviourベースAPI

`owner.GetInstanceID()`をエンティティIDとして使用する便利なラッパー。

### Register

```csharp
public void Register(MonoBehaviour owner, TData initialData)
```

MonoBehaviourのインスタンスIDを使用して登録します。

```csharp
entitySet.Register(this, new EnemyState { Health = 100 });
```

### Unregister

```csharp
public void Unregister(MonoBehaviour owner)
```

MonoBehaviour参照を使用して登録解除します。

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
- `GetDataRef(MonoBehaviour owner)`
- `UpdateData(MonoBehaviour owner, Func<TData, TData> updater)`
- `Contains(MonoBehaviour owner)`
- `NotifyDataChanged(MonoBehaviour owner)`

---

## 反復処理

### ForEach

```csharp
public void ForEach(Action<int, TData> callback)
```

すべてのエンティティを反復処理します。安全な変更のために逆方向反復を使用します。

```csharp
entitySet.ForEach((id, state) => {
    if (state.Health <= 0)
    {
        entitySet.Unregister(id);  // 反復中も安全
    }
});
```

### Dataプロパティ

パフォーマンスクリティカルなコードでは、NativeSlice経由で直接データにアクセスできます。

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

ReactiveEntitySetSOに自動登録するエンティティの基底クラス。

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
| OnBeforeUnregister | 登録解除前に呼ばれる（クリーンアップ用にオーバーライド） |

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

Sparse Setデータ構造はO(1)アクセスを提供しながら、反復処理のためにキャッシュフレンドリーな連続ストレージを維持します。

---

## スレッドセーフティ

ReactiveEntitySetSOは**スレッドセーフではありません**。すべての操作はUnityメインスレッドで実行する必要があります。Jobs、asyncコンテキスト、バックグラウンドスレッドからのアクセスは未定義動作を引き起こします。

---

## 参考資料

- [リアクティブエンティティセットガイド]({{ '/ja/guides/reactive-entity-sets' | relative_url }}) - RESの使い方
- [ランタイムセットリファレンス](runtime-set-types) - シンプルなオブジェクト追跡
- [イベントタイプリファレンス](event-types) - RESが使用するイベントチャンネル
