---
layout: default
title: Asset Browser
parent: デバッグ
nav_order: 6
---

# Asset Browser

{: .note }
> v1.1.0以降で利用可能。

## 目的

このガイドでは、すべてのReactive SOアセットを一元管理・検査するためのAsset Browser Windowの使用方法を説明します。アセットの検索、フィルタリング、およびPlay Mode中の操作方法を学びます。

---

## ウィンドウを開く

**Window > Reactive SO > Asset Browser**に移動します。

---

## 機能

Asset Browserは、すべてのReactive SOアセットタイプに対する統合ビューを提供します。

- **Event Channels**
- **Variables**
- **Runtime Sets**
- **Reactive Entity Sets**

### タブ

上部のタブを使用して、アセットカテゴリを切り替えます。

### 検索とフィルタ

- **検索バー**: 名前でアセットをフィルタリングします。
- **タイプフィルタ**: 特定の型でアセットをフィルタリングします（例：`IntEventChannelSO`のみ表示）。

### ドラッグ＆ドロップ

リストからInspectorのフィールドにアセットを直接ドラッグできます。これにより、Projectウィンドウ内を検索することなく依存関係を簡単に割り当てることができます。

---

## Play Modeでの操作

Asset BrowserはPlay Mode中にインタラクティブになります。

### Event Channels

- **Voidイベント**: **Raise**ボタンをクリックして、イベントを即座に発火させます。
- **その他のイベント**: **Raise**をクリックするとProjectウィンドウのアセットがpingされます（手動発火にはInspectorが必要です）。

### Variables

- **Value列**: 変数の現在のランタイム値を表示します。
- リアルタイムで更新されます。

### Runtime Sets

- **Count列**: 現在セットに含まれているアイテム数を表示します。

### Reactive Entity Sets

- **Count列**: 登録されているエンティティ数を表示します。

---

## ユースケース

### プロジェクトの概要

プロジェクト内で定義されているすべてのイベントと変数を素早く確認し、重複作成を防ぎます。

### イベントのデバッグ

Voidイベント（`OnGameStart`や`OnPlayerDeath`など）をリストから直接発火させて、システムの応答をテストします。

### 状態の監視

個別に選択することなく、複数の変数の値がリアルタイムで更新される様子を監視します。
