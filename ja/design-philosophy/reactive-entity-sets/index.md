---
layout: default
title: RES設計
parent: 設計思想
nav_order: 3
has_children: true
---

# Reactive Entity Sets: 設計思想

---

## 目的

このセクションでは、Reactive Entity Sets（RES）の設計思想を説明します。これらの概念を理解することで、RESをいつ、どのように効果的に使用するかを判断できるようになります。

RESは、Unityの他の状態管理アプローチとは根本的に異なります。実装に入る前に、RESが**なぜ**存在し、どのような問題を解決するのかを理解することが重要です。

---

## 学習内容

| ページ | トピック |
|--------|----------|
| [設計アプローチ](approach) | GPU InstancerやECSとの違い、RESが優先すること |
| [Entity、Object、View](entity-object-view) | EntityとObjectの区別、ViewとしてのGameObject、シーン非依存データ |
| [データガイドライン](data-guidelines) | RESに含めるべきデータ、データ/ロジック分離パターン |
| [集合論の基礎](set-theory) | 数学的基礎、Viewの種類、データベースとの対応 |

---

## 核心的な洞察

RESは重要なアーキテクチャの反転に基づいて構築されています。

**従来のUnityパターン**

```
GameObjectがデータを所有
  └── MonoBehaviourが状態を保持
      └── GameObjectと共に破棄される
```

**RESパターン**

```
ScriptableObjectがデータを所有（永続的）
  └── GameObjectはデータを表示（ビュー）
      └── 状態を失わずに破棄可能
```

この反転により、シーン間の永続化、ネットワーク同期、データと表示の明確な分離が可能になります。

---

## クイックナビゲーション

**RESが初めての方**

[設計アプローチ](approach)から始めて、RESが他のソリューションとどう異なるかを理解してください。

**実践的なガイダンスが必要な方**

[データガイドライン](data-guidelines)で、どのデータを含めるべきかのルールを確認してください。

**理論に興味がある方**

[集合論の基礎](set-theory)で数学的な基礎を探求してください。

---

## 次のステップ

設計思想を理解したら、[Reactive Entity Setsガイド]({{ '/ja/guides/reactive-entity-sets/' | relative_url }})で実践的な実装を確認してください。
