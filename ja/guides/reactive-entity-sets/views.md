---
layout: default
title: ビュー
parent: Reactive Entity Sets
grand_parent: ガイド
nav_order: 4
---

# ビュー

---

## 目的

このページでは、`ReactiveView<TData>`を使ってエンティティのサブセットを動的に管理する方法を説明します。データ条件だけのビュー、トレイト条件を含むビュー、メンバーシップイベント、ライフサイクルの各トピックを扱います。

---

## ビューとは

ビューはセット内のエンティティのうち、指定した条件を満たすものだけを追跡するオブジェクトです。エンティティのデータやトレイトが変わると、ビューは自動的にメンバーシップを再評価します。条件を満たすようになったエンティティには`OnEnter`、満たさなくなったエンティティには`OnExit`が発火します。

毎フレーム`ForEach`でセット全体を走査して絞り込む代わりに、ビューを使うと変更があったエンティティだけを再評価するため効率的です。

---

## データのみのビューを作成する

HPが0より大きいエンティティだけを追跡するビューを作ります。

```csharp
public class EnemyAIManager : MonoBehaviour
{
    [SerializeField] private EnemyEntitySetSO enemySet;

    private ReactiveView<EnemyState> aliveEnemies;

    private void OnEnable()
    {
        aliveEnemies = enemySet.CreateView(
            state => state.Health > 0
        );

        aliveEnemies.OnEnter += id => Debug.Log($"Enemy {id} is now alive in view");
        aliveEnemies.OnExit  += id => Debug.Log($"Enemy {id} left the alive view");
    }

    private void OnDisable()
    {
        aliveEnemies?.Dispose();
        aliveEnemies = null;
    }
}
```

`CreateView`を呼んだ時点で、既存の全エンティティに対して条件が評価され、初期メンバーが決まります。

---

## トレイト対応ビューを作成する

トレイトとデータの両方を条件に使う場合、`Func<TData, ulong, bool>`のオーバーロードを使います。

```csharp
// IsAggroが立っていて、かつHPが50%以上の敵
var aggroAlive = enemySet.CreateView(
    (state, traitMask) =>
    {
        ulong aggroFlag = TraitMaskUtility.ToUInt64(EnemyTraits.IsAggro);
        return (traitMask & aggroFlag) != 0 && state.Health > state.MaxHealth / 2;
    },
    ViewTrigger.All,
    TraitMaskUtility.ToUInt64(EnemyTraits.IsAggro)
);
```

第3引数`observedTraitMask`には、このビューが反応するトレイトフラグを渡します。`IsAggro`の変化に限定することで、無関係なトレイト変更での不要な再評価を防ぎます。

---

## ViewTriggerモード

`ViewTrigger`は、ビューがどのタイミングで再評価するかを制御します。

| 値 | 説明 |
|---|------|
| `None` | 再評価しない（登録・登録解除のみ追跡） |
| `DataOnly` | データ変更時に再評価 |
| `TraitsOnly` | トレイト変更時に再評価 |
| `All` | データ変更とトレイト変更の両方で再評価 |

データのみのビューには`ViewTrigger.DataOnly`（デフォルト）を使います。トレイトに反応させたい場合は`TraitsOnly`か`All`を指定します。

```csharp
// データが変わったときだけ再評価（トレイト変更は無視）
var dataView = enemySet.CreateView(
    state => state.Health > 0,
    ViewTrigger.DataOnly
);

// トレイトが変わったときだけ再評価（データ変更は無視）
var traitView = enemySet.CreateView(
    (state, mask) => (mask & TraitMaskUtility.ToUInt64(EnemyTraits.IsDead)) == 0,
    ViewTrigger.TraitsOnly,
    TraitMaskUtility.ToUInt64(EnemyTraits.IsDead)
);
```

---

## メンバーシップイベント

エンティティがビューに入るときと出るときにイベントが発火します。

```csharp
private ReactiveView<EnemyState> stunView;

private void OnEnable()
{
    stunView = enemySet.CreateView(
        (state, mask) =>
        {
            ulong stunFlag = TraitMaskUtility.ToUInt64(EnemyTraits.IsStunned);
            return (mask & stunFlag) != 0;
        },
        ViewTrigger.TraitsOnly,
        TraitMaskUtility.ToUInt64(EnemyTraits.IsStunned)
    );

    stunView.OnEnter += OnEnemyStunned;
    stunView.OnExit  += OnEnemyRecovered;
}

private void OnEnemyStunned(int entityId)
{
    // スタン演出を開始
    PlayStunEffect(entityId);
}

private void OnEnemyRecovered(int entityId)
{
    // スタン演出を停止
    StopStunEffect(entityId);
}

private void OnDisable()
{
    stunView?.Dispose();
}
```

---

## ビューメンバーをイテレートする

ビューのメンバーを`foreach`で走査します。

```csharp
private void Update()
{
    // ビューに所属するエンティティだけ処理
    foreach (int id in aliveEnemies)
    {
        if (enemySet.TryGetData(id, out var state))
        {
            UpdateEnemyAI(id, state);
        }
    }
}
```

特定のエンティティがビューに含まれているかどうかはO(1)で確認できます。

```csharp
if (aliveEnemies.Contains(targetId))
{
    // ターゲットが生存中
}

Debug.Log($"生存中の敵: {aliveEnemies.Count}");
```

---

## ライフサイクルと破棄

ビューはNativeメモリを保持するため、使い終わったら必ず`Dispose`を呼びます。`Dispose`を呼ぶとビューはセットから登録解除され、`OnEnter`/`OnExit`のイベントも解放されます。

```csharp
private ReactiveView<EnemyState> view;

private void OnEnable()
{
    view = enemySet.CreateView(state => state.Health > 0);
    view.OnEnter += HandleEnter;
}

private void OnDisable()
{
    // Disposeを忘れるとメモリリーク
    view?.Dispose();
    view = null;
}
```

---

## バルク操作時の挙動

`Clear()`や`RestoreSnapshot()`を実行すると、ビューのメンバーシップはサイレントに再構築されます。この場合、`OnEnter`と`OnExit`は発火しません。UI要素のリフレッシュなど、バルク操作後に一括更新が必要な場合は`Count`や`Contains`で状態を確認してください。

```csharp
// バルク操作後に手動でUIを同期
enemySet.Clear();
RefreshAllUIFromScratch();  // OnExit/OnEnterを待たず即時同期
```

---

## APIサマリー

### ビューの作成

| メソッド | 説明 |
|---------|------|
| `CreateView(Func<TData, bool> predicate, ViewTrigger triggerOn)` | データのみのビューを作成 |
| `CreateView(Func<TData, ulong, bool> predicate, ViewTrigger triggerOn, ulong observedTraitMask)` | トレイト対応ビューを作成 |

### ReactiveView<TData>

| メンバー | 説明 |
|---------|------|
| `bool Contains(int id)` | メンバーかどうかをO(1)で確認 |
| `int Count` | 現在のメンバー数 |
| `GetEnumerator()` | `NativeHashSet<int>`のEnumeratorを取得 |
| `event Action<int> OnEnter` | エンティティがビューに入ったとき発火 |
| `event Action<int> OnExit` | エンティティがビューから出たとき発火 |
| `void Dispose()` | ビューを破棄してリソースを解放 |

---

## パフォーマンス特性

| 操作 | 時間計算量 |
|------|-----------|
| `Contains` | O(1) |
| メンバーシップ再評価（1エンティティ） | O(1) |
| 初期化時の全評価 | O(n) |
| バルク操作後の再構築 | O(n) |

ビューのメンバーシップはNativeHashSetで管理されます。登録済みのビュー数が増えるほど、データ変更やトレイト変更時のオーバーヘッドも増えます。不要になったビューはすぐに`Dispose`してください。

---

## 次のステップ

- [パターン](patterns) - 一般的な使用パターンを確認できます
- [ベストプラクティス](best-practices) - パフォーマンスとトラブルシューティングについて解説しています
