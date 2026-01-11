---
layout: default
title: Monitor
parent: デバッグ
nav_order: 2
---

# Monitor Window

{: .note }
> 統合されたMonitor Windowはv2.1.0から利用可能です。v2.1.0以前は、Event Channels、Variables、Runtime Sets用の個別のモニターウィンドウが存在していました。

## 目的

このガイドでは、イベント、変数、セットをリアルタイムで追跡するためのMonitor Windowの使用方法を説明します。Event Monitor、Variable Monitor、Runtime Set Monitorの機能が1つのタブ付きインターフェースに統合されています。

---

## ウィンドウを開く

**Window > Reactive SO > Monitor**に移動します。

---

## 共通機能

Monitor Windowはすべてのタブで共通のインターフェースを持っています。

### ツールバー

- **Clear**: 現在のログをすべて削除します。
- **Search**: 名前、型、その他のテキストでログリストをフィルタリングします。
- **メニュー (⋮)**
    - **Export/CSV**: 現在のログをCSVファイルに保存します。
    - **Export/CSV (Excel)**: Excel互換性のためにUTF-8 BOM付きで保存します。
    - **Max Entries**: 保持する最大ログ数（100 - 10K）を設定します。

### フッター

- **Status**: 総ログ数と現在表示（フィルタリング）されている数を表示します。
- **[LIVE]**: リアルタイム更新がアクティブ（Play Mode中）であることを示します。

### ログの永続性

ログは以下のように動作します。

| アクション | 動作 |
|------------|------|
| Play Modeに入る | ログは自動的にクリアされる |
| Play Mode中 | イベントと変更が記録される |
| Play Modeを抜ける | ログはレビュー用に保持される |
| 次のPlay Mode | 前回のログはクリアされる |

---

## Event Channels タブ

`EventChannelSO` アセットによって発火されたイベントを追跡します。

### 列

| 列 | 説明 |
|----|------|
| Time | Play Mode開始からの経過秒数 |
| Name | イベントチャンネルアセットの名前 |
| Type | イベントタイプ（Void、Int、Floatなど） |
| Value | イベントと共に渡された値 |
| Listeners | アクティブなリスナー数 |
| Caller | イベントを発火させたコードの場所 |

### 呼び出し元情報

Caller列には `FileName.cs:MethodName:LineNumber` が表示されます。クリックするとIDEでファイルが開きます。

{: .note }
> **制限事項**: 呼び出し元情報はコードからの呼び出しでのみ機能します。UnityEvents（例：Inspectorのボタンクリック）経由で発火されたイベントは "-" と表示されます。

---

## Actions タブ

`ActionSO` アセットの実行を追跡します。

### 列

| 列 | 説明 |
|----|------|
| Time | Play Mode開始からの経過秒数 |
| Name | アクションアセットの名前 |
| Type | アクションタイプ |
| Description | アクションの説明（Inspectorから） |
| Caller | アクションを実行したコードの場所 |

---

## Variables タブ

`VariableSO` アセットの値の変更を追跡します。

### 列

| 列 | 説明 |
|----|------|
| Time | Play Mode開始からの経過秒数 |
| Name | 変数アセットの名前 |
| Type | 変数タイプ（Int、Float、Boolなど） |
| Old Value | 変更前の値 |
| New Value | 変更後の値 |

値が実際に変更された場合（`EqualityComparer<T>`を使用）のみログが記録されます。

---

## Runtime Sets タブ

`RuntimeSetSO` アセットに対する操作を追跡します。

### 列

| 列 | 説明 |
|----|------|
| Time | Play Mode開始からの経過秒数 |
| Name | ランタイムセットアセットの名前 |
| Type | アイテムタイプ（GameObject、Transformなど） |
| Operation | 実行されたアクション（Add、Remove、Clear） |
| Details | 関連するアイテムの情報 |

オブジェクトの登録および登録解除のライフサイクルをデバッグするために使用します。

---

## Reactive Entity Sets タブ

`ReactiveEntitySetSO` アセットに対する操作を追跡します。

### 列

| 列 | 説明 |
|----|------|
| Time | Play Mode開始からの経過秒数 |
| Name | エンティティセットアセットの名前 |
| Data Type | 状態構造体の型 |
| Operation | アクション（Register、Unregister、SetData、UpdateData） |
| Details | エンティティIDと操作の詳細 |

エンティティのライフサイクルと状態更新を監視するために使用します。

---

## ログのエクスポート

任意のタブからCSVにログをエクスポートして、外部で分析できます。

1. エクスポートしたいタブを選択します。
2. ツールバーのケバブメニュー（**⋮**）をクリックします。
3. **Export/CSV** または **Export/CSV (Excel)** を選択します。
4. 保存場所を選択します。

---

## 参照

- [デバッグ概要]({{ '/ja/debugging/' | relative_url }})
- [トラブルシューティング]({{ '/ja/troubleshooting' | relative_url }})
