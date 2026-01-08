---
layout: default
title: 設計思想
nav_order: 3
has_children: true
---

# 設計思想

---

## 目的

このセクションでは、Reactive SOの基盤となる概念を説明します。これらの原則を理解することで、より良いアーキテクチャ上の判断ができ、各機能を効果的に使用できるようになります。

---

## 基本原則

Reactive SOは一つの基本的なアイデアに基づいています。**ScriptableObjectはグローバルな共有リソースである**

これは以下を意味します。

- すべてのシーンを跨いで永続する
- 同じアセットを参照するすべてのスクリプトが同じインスタンスを共有する
- グローバルな状態やシステム間通信に最適

この原則を理解することが、各状況で適切なツールを選択するために重要です。

---

## 学習内容

| ページ | トピック |
|--------|----------|
| [ScriptableObjectの基礎](scriptableobject-basics) | グローバル共有リソース、Entity vs Object、Views としての GameObject、シーン永続性 |
| [アーキテクチャパターン](architecture-patterns) | 4つのツールの比較、Instance vs Global ルール、共通パターン、決定ガイド |
| [テスト容易性](testability) | Reactive SOがテスト可能な理由、DI比較、モックライブラリなしのテスト |

---

## クイックナビゲーション

**Reactive SOを初めて使う方**

まず[はじめに]({{ '/ja/getting-started' | relative_url }})でインストールと基本的な使い方を学び、その後ここに戻って理解を深めてください。

**すでにReactive SOを使用している方**

[アーキテクチャパターン](architecture-patterns)で適切なツールを選択するための決定ガイドをご覧ください。
