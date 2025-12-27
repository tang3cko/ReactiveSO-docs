---
layout: default
title: ランタイムセットタイプ
parent: リファレンス
nav_order: 3
---

# ランタイムセットタイプ

## 目的

このリファレンスでは、ビルトインのランタイムセットタイプをすべて解説します。動的オブジェクトコレクション管理のためのAPIリファレンス、ユースケース、コード例を確認できます。

---

## タイプ概要

| タイプ | 要素タイプ | 特別な機能 |
|------|--------------|------------------|
| GameObjectRuntimeSetSO | `GameObject` | `DestroyItems()`メソッド |
| TransformRuntimeSetSO | `Transform` | - |

---

## 共通API

すべてのランタイムセットタイプは以下のAPIを共有します。

### プロパティ

| プロパティ | タイプ | 説明 |
|----------|------|-------------|
| Items | `IReadOnlyList<T>` | すべてのアイテムへの読み取り専用アクセス |
| Count | `int` | セット内のアイテム数 |

### メソッド

| メソッド | 戻り値 | 説明 |
|--------|---------|-------------|
| `Add(T item)` | `void` | アイテムを追加（重複は無視） |
| `Remove(T item)` | `bool` | アイテムを削除、成功を返す |
| `Clear()` | `void` | すべてのアイテムを削除 |
| `Contains(T item)` | `bool` | アイテムがセット内にあるか確認 |

### イベント

| イベント | タイプ | 説明 |
|-------|------|-------------|
| onItemsChanged | `VoidEventChannelSO` | アイテム追加/削除時に発火 |
| onCountChanged | `IntEventChannelSO` | 新しいカウントで発火 |

---

## GameObjectRuntimeSetSO

GameObject参照を追跡します。最も一般的に使用されるランタイムセットタイプです。

### 作成

```text
Create > Reactive SO > Runtime Sets > GameObject Runtime Set
```

### 基本的な使い方

```csharp
[SerializeField] private GameObjectRuntimeSetSO activeEnemies;

// 登録（敵スクリプト内）
private void OnEnable()
{
    activeEnemies?.Add(gameObject);
}

private void OnDisable()
{
    activeEnemies?.Remove(gameObject);
}
```

### 反復処理

```csharp
// すべてのアイテムにアクセス
foreach (var enemy in activeEnemies.Items)
{
    if (enemy != null)
    {
        // 敵を処理
    }
}

// カウントを確認
if (activeEnemies.Count == 0)
{
    OnAllEnemiesDefeated();
}
```

### DestroyItemsメソッド

GameObjectRuntimeSetSOには、追跡されているすべてのオブジェクトを破棄する特別なメソッドが含まれています。

```csharp
// すべてのアイテムを破棄してセットをクリア
activeEnemies.DestroyItems();
```

ウェーブベースのゲームやクリーンアップシナリオに便利です。

### 一般的なユースケース

- シーン内のアクティブな敵
- 収集可能なピックアップ
- インタラクト可能なオブジェクト
- 飛行中の発射物
- スポーンされたアイテム

---

## TransformRuntimeSetSO

Transform参照を追跡します。完全なGameObjectオーバーヘッドなしでトランスフォームデータが必要な場合に使用します。

### 作成

```text
Create > Reactive SO > Runtime Sets > Transform Runtime Set
```

### 基本的な使い方

```csharp
[SerializeField] private TransformRuntimeSetSO waypoints;

// 登録
private void OnEnable()
{
    waypoints?.Add(transform);
}

private void OnDisable()
{
    waypoints?.Remove(transform);
}
```

### 反復処理

```csharp
// 最も近いウェイポイントを見つける
Transform nearest = null;
float nearestDist = float.MaxValue;

foreach (var waypoint in waypoints.Items)
{
    if (waypoint == null) continue;

    float dist = Vector3.Distance(transform.position, waypoint.position);
    if (dist < nearestDist)
    {
        nearestDist = dist;
        nearest = waypoint;
    }
}
```

### 一般的なユースケース

- パスファインディング用のウェイポイント
- スポーンポイント
- パトロールルートノード
- カメラターゲット
- 参照ポイント

---

## 変更への購読

イベントチャンネルを使用してコレクションの変更に反応します。

### セットアップ

1. `onItemsChanged`用のVoidEventChannelSOを作成
2. `onCountChanged`用のIntEventChannelSOを作成
3. Inspectorでランタイムセットに割り当て

### コード

```csharp
[SerializeField] private IntEventChannelSO onEnemyCountChanged;

private void OnEnable()
{
    onEnemyCountChanged.OnEventRaised += HandleCountChange;
}

private void OnDisable()
{
    onEnemyCountChanged.OnEventRaised -= HandleCountChange;
}

private void HandleCountChange(int count)
{
    enemyCountText.text = $"敵: {count}";
}
```

---

## 重複処理

ランタイムセットは自動的に重複を防ぎます。既存のアイテムで`Add()`を呼び出しても何も起こりません。

```csharp
enemies.Add(enemy);  // 追加（カウント: 1）
enemies.Add(enemy);  // 無視（カウント: 1）
enemies.Add(enemy);  // 無視（カウント: 1）
```

---

## カスタムタイプの作成

`RuntimeSetSO<T>`を継承してカスタムランタイムセットタイプを作成できます。

```csharp
using Tang3cko.ReactiveSO;
using UnityEngine;

[CreateAssetMenu(
    fileName = "EnemyRuntimeSet",
    menuName = "Reactive SO/Runtime Sets/Enemy Runtime Set"
)]
public class EnemyRuntimeSetSO : RuntimeSetSO<Enemy>
{
    // 基本機能をすべて継承

    // 必要に応じてカスタムメソッドを追加
    public Enemy GetRandomEnemy()
    {
        if (Count == 0) return null;
        return Items[Random.Range(0, Count)];
    }
}
```

---

## ベストプラクティス

### 登録パターン

常に`OnEnable`で登録し、`OnDisable`で登録解除します。

```csharp
// ✅ Good: OnEnable/OnDisableペア
private void OnEnable() => enemies?.Add(gameObject);
private void OnDisable() => enemies?.Remove(gameObject);

// ❌ Bad: Start/OnDestroyペア
private void Start() => enemies?.Add(gameObject);
private void OnDestroy() => enemies?.Remove(gameObject);
```

`OnDestroy`を使用すると、オブジェクトが破棄されずに無効化された場合に孤立したエントリが残る可能性があります。

### null条件演算子

セットが割り当てられていない可能性がある場合は`?.`を使用します。

```csharp
// ✅ Good: 割り当てられていない場合も安全
activeEnemies?.Add(gameObject);

// ❌ Bad: 割り当てられていない場合NullReferenceException
activeEnemies.Add(gameObject);
```

---

## 参考資料

- [ランタイムセットガイド]({{ '/ja/guides/runtime-sets' | relative_url }}) - ランタイムセットの使い方
- [ランタイムセットモニター]({{ '/ja/debugging/runtime-set-monitor' | relative_url }}) - セット内容のデバッグ
- [デバッグ概要]({{ '/ja/debugging/' | relative_url }}) - すべてのデバッグツール
