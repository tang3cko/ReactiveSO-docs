---
layout: default
title: 基本的な使い方
parent: Reactive Entity Sets
grand_parent: ガイド
nav_order: 1
---

# 基本的な使い方

---

## 目的

このページでは、プロジェクトでReactive Entity Setsをセットアップして使用する手順を説明します。state構造体、entity setアセット、entityコンポーネントを作成します。

---

## ステップ1: state構造体を定義する

エンティティデータを保持する構造体を作成します。`[System.Serializable]`である必要があります。

```csharp
using System;

[Serializable]
public struct EnemyState
{
    public int Health;
    public int MaxHealth;
    public bool IsStunned;

    public float HealthPercent => MaxHealth > 0 ? (float)Health / MaxHealth : 0f;
    public bool IsDead => Health <= 0;
}
```

---

## ステップ2: reactive entity setアセットを作成する

`ReactiveEntitySetSO<T>`を継承するクラスを作成します。

```csharp
using Tang3cko.ReactiveSO;
using UnityEngine;

[CreateAssetMenu(
    fileName = "EnemyEntitySet",
    menuName = "Reactive SO/Entity Sets/Enemy"
)]
public class EnemyEntitySetSO : ReactiveEntitySetSO<EnemyState>
{
    // ベースクラスがすべての機能を提供
}
```

次にProjectウィンドウで以下のメニューパスを選択してアセットを作成します。

```text
Create > Reactive SO > Entity Sets > Enemy
```

---

## ステップ3: イベントチャンネルを作成する（オプション）

セットレベルの変更通知が必要な場合、イベントチャンネルを作成します。

```text
Create > Reactive SO > Channels > Int Event
```

entity setのフィールドに割り当てます。利用可能なイベントフィールドは以下の通りです。

- **On Item Added** - エンティティが登録されたときに発火
- **On Item Removed** - エンティティが登録解除されたときに発火
- **On Data Changed** - いずれかのエンティティのデータが変更されたときに発火
- **On Set Changed** - 任意の変更時に発火

---

## ステップ4: entityコンポーネントを作成する

自動ライフサイクル管理のために`ReactiveEntity<T>`ベースクラスを使用します。

```csharp
using Tang3cko.ReactiveSO;
using UnityEngine;

public class Enemy : ReactiveEntity<EnemyState>
{
    [SerializeField] private EnemyEntitySetSO entitySet;
    [SerializeField] private int maxHealth = 100;

    // 必須: 使用するセットを指定
    protected override ReactiveEntitySetSO<EnemyState> Set => entitySet;

    // 必須: 初期状態を指定
    protected override EnemyState InitialState => new EnemyState
    {
        Health = maxHealth,
        MaxHealth = maxHealth,
        IsStunned = false
    };

    public void TakeDamage(int damage)
    {
        var state = State;
        state.Health = Mathf.Max(0, state.Health - damage);
        State = state;  // 自動的にイベントをトリガー

        if (state.IsDead)
        {
            Destroy(gameObject);  // OnDisableで自動的に登録解除
        }
    }
}
```

---

## ステップ5: エンティティデータをクエリする

任意のスクリプトからエンティティデータにアクセスします。

```csharp
public class EnemyManager : MonoBehaviour
{
    [SerializeField] private EnemyEntitySetSO entitySet;

    public int GetTotalEnemyHealth()
    {
        int total = 0;
        entitySet.ForEach((id, state) => {
            total += state.Health;
        });
        return total;
    }
}
```

---

## APIリファレンス

### プロパティ

| プロパティ | 型 | 説明 |
|-----------|-----|------|
| `Count` | `int` | 登録されたエンティティ数 |
| `EntityIds` | `ArraySegment<int>` | すべての登録エンティティID |
| `Data` | `ArraySegment<TData>` | すべてのエンティティデータ |

### メソッド

| メソッド | 説明 |
|---------|------|
| `Register(owner, data)` | 初期状態でエンティティを登録 |
| `Unregister(owner)` | セットからエンティティを削除 |
| `GetData(owner)` | エンティティの現在の状態を取得 |
| `TryGetData(owner, out data)` | エンティティの状態を安全に取得 |
| `SetData(owner, data)` | エンティティの状態を更新 |
| `UpdateData(owner, func)` | 関数で状態を更新 |
| `GetDataRef(id)` | 直接変更用に状態への参照を取得 |
| `NotifyDataChanged(owner)` | GetDataRef変更後に手動で通知 |
| `Contains(owner)` | エンティティが存在するか確認 |
| `Clear()` | すべてのエンティティを削除 |
| `ForEach(action)` | すべてのエンティティを反復 |
| `SubscribeToEntity(id, callback)` | エンティティの変更をサブスクライブ |
| `UnsubscribeFromEntity(id, callback)` | エンティティからサブスクライブ解除 |

---

## パフォーマンス特性

Reactive Entity SetsはSparse Setデータ構造を使用します。

| 操作 | 時間計算量 |
|------|-----------|
| Register | O(1) |
| Unregister | O(1) |
| GetData | O(1) |
| SetData | O(1) |
| イテレーション | O(n) |

Sparse Setはページ単位でメモリを割り当て、使用されるID範囲のページのみを作成します。これはスパースなID分布に効率的です。

---

## 次のステップ

- [イベント](events) - エンティティごとおよびセットレベルのサブスクリプションについて学ぶ
- [パターン](patterns) - 一般的な使用パターンを見る
