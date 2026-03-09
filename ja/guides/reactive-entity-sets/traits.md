---
layout: default
title: トレイト
parent: Reactive Entity Sets
grand_parent: ガイド
nav_order: 3
---

# トレイト

{: .warning }
> **実験的機能** - Traitsはv2.2.0（未リリース）で利用可能です。APIは将来のバージョンで変更される可能性があります。本番環境での使用は自己責任で行ってください。

---

## 目的

このページでは、エンティティにビットフラグのTraitsを付与して管理する方法を説明します。Traitsの定義、変更、照会、フィルタリングイテレーションの各操作を順番に示します。

---

## トレイトとは

Traitsは1つのエンティティに付与するビットフラグの集合です。内部では64ビットの`ulong`マスクとして格納されるため、GCアロケーションを発生させません。`IsDead`や`IsStunned`といった状態をデータ構造体に含めずに管理できます。

---

## ステップ1 — Traitsのenumを定義する

`[Flags]`属性を付けた`enum`を作成します。各メンバーは2のべき乗にします。

```csharp
[Flags]
public enum EnemyTraits
{
    None      = 0,
    IsAggro   = 1 << 0,
    IsDead    = 1 << 1,
    IsStunned = 1 << 2,
}
```

---

## ステップ2 — Traitsを変更する

エンティティが登録済みであることを前提として、以下のメソッドでTraitsを操作します。

```csharp
// IsAggroを追加（既存のTraitsはそのまま）
enemySet.AddTraits(entityId, EnemyTraits.IsAggro);

// IsStunnedを削除
enemySet.RemoveTraits(entityId, EnemyTraits.IsStunned);

// Traitsを全て置き換え
enemySet.SetTraits(entityId, EnemyTraits.IsDead);

// 全Traitsを削除（マスクを0にリセット）
enemySet.ClearTraits(entityId);
```

`AddTraits`は既存のマスクにビットORで追加します。`RemoveTraits`はビットAND-NOTで削除します。`SetTraits`は既存のマスクを引数の値で上書きします。

---

## ステップ3 — Traitsを照会する

```csharp
// IsAggroとIsStunnedが両方立っているか
bool bothSet = enemySet.HasTraits(entityId, EnemyTraits.IsAggro | EnemyTraits.IsStunned);

// IsAggroまたはIsStunnedのどちらかが立っているか
bool anySet = enemySet.HasAnyTrait(entityId, EnemyTraits.IsAggro | EnemyTraits.IsStunned);

// 現在のTraitsを取得
EnemyTraits current = enemySet.GetTraits<EnemyTraits>(entityId);

// 安全な取得（エンティティが存在しない場合でも例外を投げない）
if (enemySet.TryGetTraits<EnemyTraits>(entityId, out var traits))
{
    Debug.Log(traits);
}
```

---

## ステップ4 — Traitsでフィルタリングしてイテレートする

Traitsに一致するエンティティだけを対象に処理します。

```csharp
// IsDeadが立っているエンティティのみ処理
enemySet.WithTraits(EnemyTraits.IsDead, (id, state) =>
{
    Debug.Log($"Dead enemy {id}: HP={state.Health}");
});

// IsAggroまたはIsStunnedが立っているエンティティを処理
enemySet.WithAnyTraits(EnemyTraits.IsAggro | EnemyTraits.IsStunned, (id, state) =>
{
    HandleActiveEnemy(id, state);
});

// 件数だけ取得したい場合
int deadCount = enemySet.CountWithTraits(EnemyTraits.IsDead);
int activeCount = enemySet.CountWithAnyTraits(EnemyTraits.IsAggro | EnemyTraits.IsStunned);
```

`WithTraits`と`WithAnyTraits`はどちらもO(n)でセット全体を走査します。コールバック内でエンティティを登録解除しても安全です（後ろから前方向にイテレートするため）。

---

## Traitsイベント

Traitsが変更されたとき、セットに割り当てた`IntEventChannelSO`が発火します。

| フィールド | 発火タイミング |
|-----------|---------------|
| On Trait Added | Traitsが追加されたとき |
| On Trait Removed | Traitsが削除されたとき |

Inspectorでイベントチャンネルを作成して割り当てます。

```text
Create > Reactive SO > Channels > Int Event
```

コールバックはエンティティIDを受け取ります。

```csharp
[SerializeField] private IntEventChannelSO onTraitAdded;

private void OnEnable()
{
    onTraitAdded.OnEventRaised += HandleTraitAdded;
}

private void OnDisable()
{
    onTraitAdded.OnEventRaised -= HandleTraitAdded;
}

private void HandleTraitAdded(int entityId)
{
    if (enemySet.HasTraits(entityId, EnemyTraits.IsDead))
    {
        TriggerDeathAnimation(entityId);
    }
}
```

---

## 型ロック

1つのセットで使えるTraitsのenum型は1種類だけです。最初に`AddTraits<TTraits>`や`HasTraits<TTraits>`を呼び出した時点で、その型がセットにロックされます。異なるenum型でメソッドを呼び出すと`InvalidOperationException`が発生します。

```csharp
// 最初の呼び出しでEnemyTraitsがロックされる
enemySet.AddTraits(id, EnemyTraits.IsAggro);

// 別の型を使うと例外
enemySet.AddTraits(id, SomeOtherEnum.Flag); // InvalidOperationException
```

複数の概念を管理したい場合は、それぞれ別のenumメンバーとして定義してください。

---

## APIサマリー

### 変更

| メソッド | 説明 |
|---------|------|
| `AddTraits<TTraits>(id, traits)` | 既存のTraitsにビットORで追加 |
| `RemoveTraits<TTraits>(id, traits)` | ビットAND-NOTで削除 |
| `SetTraits<TTraits>(id, traits)` | 全Traitsを置き換え |
| `ClearTraits(id)` | マスクを0にリセット |

### 照会

| メソッド | 説明 |
|---------|------|
| `HasTraits<TTraits>(id, traits)` | 指定フラグが全て立っているか |
| `HasAnyTrait<TTraits>(id, traits)` | 指定フラグのいずれかが立っているか |
| `GetTraits<TTraits>(id)` | 現在のTraitsを取得 |
| `TryGetTraits<TTraits>(id, out traits)` | 例外なしで安全に取得 |

### フィルタリングイテレーション

| メソッド | 説明 |
|---------|------|
| `WithTraits<TTraits>(traits, callback)` | 全フラグ一致のエンティティをイテレート |
| `WithAnyTraits<TTraits>(traits, callback)` | いずれかのフラグ一致のエンティティをイテレート |
| `CountWithTraits<TTraits>(traits)` | 全フラグ一致のエンティティ数 |
| `CountWithAnyTraits<TTraits>(traits)` | いずれかのフラグ一致のエンティティ数 |

---

## パフォーマンス特性

TraitsマスクはNativeArrayに連続して格納されるため、イテレーション時のキャッシュ効率は高いです。

| 操作 | 時間計算量 |
|------|-----------|
| AddTraits / RemoveTraits / SetTraits / ClearTraits | O(1) |
| HasTraits / HasAnyTrait / GetTraits | O(1) |
| WithTraits / WithAnyTraits | O(n) |
| CountWithTraits / CountWithAnyTraits | O(n) |

メモリは1エンティティあたり8バイト（`ulong`1つ）です。TraitsストレージはTraits APIを初めて呼び出したときに遅延初期化されます。

---

## 次のステップ

- [Views](views) - Traitsとデータに基づくリアクティブなエンティティサブセットについて学べます
- [パターン](patterns) - 一般的な使用パターンを確認できます
