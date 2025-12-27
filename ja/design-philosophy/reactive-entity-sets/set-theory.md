---
layout: default
title: 集合論の基礎
parent: RES設計
grand_parent: 設計思想
nav_order: 4
---

# 集合論の基礎

---

## 目的

このページでは、Reactive Entity Setsの数学的基礎を説明します。これらの概念を理解することで、RESがなぜそのように動作するのかについてより深い洞察が得られ、Viewなどの高度な機能への準備ができます。

---

## 数学的集合としてのReactiveEntitySet

ReactiveEntitySetは、各要素が（Entity ID, Data）のペアである数学的集合Sとして理解できます。

```
S = { (id₁, data₁), (id₂, data₂), ..., (idₙ, dataₙ) }
```

### 主要な性質

| 性質 | 説明 |
|------|------|
| **一意性** | 各エンティティIDは最大1回のみ出現 |
| **所属** | エンティティは集合に属するか属さないか（部分的な所属はない） |
| **データ関連付け** | 各メンバーは正確に1つの関連データ値を持つ |

---

## 演算と閉包

### 閉包性

集合族Sは、演算Oを適用してもS内の別の集合が生成される場合、演算Oに対して**閉じている**と言います。

```
∀O ∈ Operations, ∀S ∈ S: O(S) ∈ S
```

この性質により、ReactiveEntitySetへの演算は予測可能で明確に定義された結果を生成します。

### ReactiveEntitySetの演算

| 演算 | 入力 | 出力 | 閉包を維持 |
|------|------|------|-----------|
| Register | S, (id, data) | S ∪ {(id, data)} | はい |
| Unregister | S, id | S \ {(id, _)} | はい |
| SetData | S, id, data' | 更新されたデータを持つS | はい |
| Filter (View) | S, 述語 | S' ⊆ S | はい |

すべての演算は閉包を維持します。結果は常に有効なReactiveEntitySet（またはそのサブセット）です。

---

## View理論

Viewは述語によって定義されるReactiveEntitySetのサブセットです。2つの基本的なタイプがあります。

### 静的View R(c₀)

静的ビューはエンティティデータのみに依存します。

```
R(c₀) = { (id, data) ∈ S | P(data) = true }
```

ここでPは述語関数: `P: TData → bool`

**例**

```csharp
// 体力が30未満のすべての敵
var lowHealthView = enemies.CreateView(state => state.Health < 30);
```

**特徴**

- 述語は作成時に固定
- エンティティデータが変更されるとView所属が自動更新
- 外部コンテキスト不要

### 動的View R(c₁)

動的ビューはエンティティデータと外部コンテキストの両方に依存します。

```
R(c₁) = { (id, data) ∈ S | P(data, context) = true }
```

ここでPは述語関数: `P: (TData, TContext) → bool`

**例**

```csharp
// 指定位置から10ユニット以内のすべての敵
var inRangeView = enemies.CreateView<Vector3>(
    (state, position) => Vector3.Distance(state.Position, position) < 10f
);

// 特定のコンテキストで評価
var nearbyEnemies = inRangeView.Evaluate(playerPosition);
```

**特徴**

- 述語の評価にコンテキストが必要
- エンティティデータの変更は自動再評価をトリガー
- コンテキストの変更は明示的な再評価が必要

### Viewの「Reactive」

R(c₀)とR(c₁)の両方は、エンティティ状態の変更に対して**Reactive**です。

```
エンティティ状態変更 → 述語評価 → View所属更新 → 通知
```

違いはリアクティビティについてではなく、述語が外部コンテキストを必要とするかどうかです。

---

## 構造的閉包

### 定義

構造的閉包は、Viewへの演算が親集合に「逃げ出す」ことができないことを保証します。

```
View V = σ[P](S)

VはSのサブセットとして決定論的に定義
Vへの演算はV内に留まる
V外の要素にアクセスする経路は存在しない
```

### 安全なAPI設計

この原則がAPI設計を導きます。

```csharp
var view = enemies.CreateView(state => state.Health < 30);

// ✅ View内へのアクセス
view.EntityIds
view.Count
view.ForEach(...)

// ❌ 親集合への逆流なし（APIが提供されない）
// view.ParentSet  ← これは存在すべきでない
```

親集合への経路を公開しないことで、Viewで操作するコードがスコープ外のエンティティに誤ってアクセスすることを防ぎます。

---

## パフォーマンスへの影響

### 従来のアプローチ

```
クエリ → すべてのエンティティを走査 → フィルタ → 結果
コスト: クエリあたりO(n)
```

### Reactive Viewアプローチ

```
エンティティ状態変更
  → viewの述語を評価
  → viewから追加/削除
  → サブスクライバーに通知

コスト: 変更あたりO(v)、ここでv = viewの数
```

---

## 形式的定義

厳密な数学的定式化に興味がある方向け。

### ReactiveEntitySet

```
ReactiveEntitySet<TData> = (S, Σ, δ, notify)

ここで:
  S: (id, data)ペアの集合
  Σ: 演算の集合 {Register, Unregister, SetData}
  δ: S × Σ → S (遷移関数)
  notify: S × Σ → Events (通知関数)
```

### 静的View R(c₀)

```
R(c₀) = σ[P](S) = { e ∈ S | P(e.data) }

ここで:
  P: TData → bool (述語)
  σ: 選択演算子
```

### 動的View R(c₁)

```
R(c₁)(ctx) = σ[P(_, ctx)](S) = { e ∈ S | P(e.data, ctx) }

ここで:
  P: (TData, TContext) → bool (パラメータ化された述語)
  ctx: TContext (評価コンテキスト)
```

---

## まとめ

| 概念 | 説明 |
|------|------|
| 集合意味論 | RESは一意のIDを持つ数学的集合 |
| 閉包 | すべての演算が有効な集合を生成 |
| 静的View R(c₀) | データのみでフィルタ |
| 動的View R(c₁) | データ + コンテキストでフィルタ |
| Reactive性 | エンティティ変更時にViewが自動更新 |
| 構造的閉包 | Viewは親集合に逃げ出せない |

---

## 実装状況

{: .note }
> Viewは現在設計段階であり、まだ実装されていません。ここで説明されている概念は計画されているアーキテクチャを表しています。

---

## 次のステップ

- [Reactive Entity Setsガイド]({{ '/ja/guides/reactive-entity-sets/' | relative_url }}) - 実装の詳細
- [設計アプローチ](approach) - ハイレベルな思想を確認
