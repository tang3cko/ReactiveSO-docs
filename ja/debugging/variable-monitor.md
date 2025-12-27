---
layout: default
title: 変数モニター（Variable Monitor）
parent: デバッグ
nav_order: 3
---

# Variable monitor

{: .note }
> v1.1.0以降で利用可能。

## 目的

このガイドでは、変数の値をリアルタイムで追跡するためのVariable Monitor Windowの使用方法を説明します。現在の値の確認、型によるフィルタリング、デバッグ中の変数アセットの素早い特定方法を学びます。

---

## ウィンドウを開く

**Window > Reactive SO > Variable Monitor**に移動します。

<!-- TODO: Add screenshot of Variable Monitor Window showing list of variables with their current values -->

---

## ウィンドウの列

| 列 | 説明 |
|--------|-------------|
| Name | 変数アセットの名前 |
| Type | 変数の型 (Int、Float、Boolなど) |
| Value | Play Mode中の現在の値 |
| Path | プロジェクト内のアセットパス |

---

## 機能

### 型フィルタリング

型ドロップダウンを使用して変数を型でフィルタリングします。Int、Float、Bool、String、Vector2、Vector3、Color、Quaternionなどのフィルターが利用可能です。

### 検索

検索ボックスに入力して変数を名前でフィルタリングします。部分一致が機能するため、「Health」と入力すると`PlayerHealth`と`EnemyHealth`の両方が表示されます。

### アセットをPing

変数の行をクリックすると、Projectウィンドウでアセットがping（ハイライト）されます。これにより、検査のためにアセットをすばやく見つけて選択できます。

### フォーマット表示

特殊な値の型は読みやすい形式で表示されます。

| 型 | 表示形式 |
|------|----------------|
| Vector2 | (x, y) |
| Vector3 | (x, y, z) |
| Quaternion | (x, y, z, w) |
| Color | RGBAとカラープレビュー |
| Bool | True/False |

---

## ユースケース

### ゲーム状態の確認

Play Mode中にVariable Monitorを開いて、すべての変数値を一覧で確認できます。これは、Inspectorで各アセットを個別に選択するよりも高速です。

### 予期しない値の発見

ゲームの動作がおかしい場合、Variable Monitorで期待と一致しない値を確認します。例えば、プレイヤーのダメージがおかしい場合、`PlayerDamageMultiplier`が期待される値を示しているかどうかを確認します。

### 初期化の検証

シーンロード後、変数が期待される初期値を持っていることを確認します。初期値が不正な変数は、`ResetToInitial()`呼び出しが不足していることを示していることが多いです。

### クロスシーン状態のデバッグ

変数はシーンをまたいで永続するため、Variable Monitorを使用してシーン遷移中に値が正しく引き継がれることを確認します。

---

## リアルタイム更新

Play Mode中は値が自動的に更新されます。変数の値が変更されるとウィンドウが更新されるため、手動で更新しなくても現在の状態を確認できます。

Play Mode外では、値はシリアライズされたアセットの状態（通常は初期値）を反映します。

---

## ヒント

- プレイテスト中はGame viewの隣にウィンドウをドッキングしておく
- 型フィルタリングを使用して特定の変数カテゴリに焦点を当てる
- 行をクリックしてアセットをpingし、InspectorでGPU Sync設定を確認
- Event Monitorと組み合わせて、値の変更とイベントを関連付ける

---

## 参考資料

- [変数ガイド]({{ '/ja/guides/variables' | relative_url }}) - 変数の使い方
- [デバッグ概要]({{ '/ja/debugging/' | relative_url }}) - すべてのデバッグツール
- [イベントモニター](event-monitor) - リアルタイムイベント追跡
