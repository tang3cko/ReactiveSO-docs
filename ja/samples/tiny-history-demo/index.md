---
layout: default
title: Tiny History デモ
parent: サンプル
nav_order: 4
has_children: true
---

# Tiny History デモ

---

## 目的

このサンプルは、Event Channels と Reactive Entity Sets が本番レベルのアプリケーションでどのように連携するかを示します。Reactive SO を使用して複雑なシステムを構築する際のリファレンスとしてご活用ください。

> **上級者向けショーケース** - これはすべての機能が連携して動作する統合サンプルです。Reactive SO を初めて使う方は、まずシンプルなサンプル（[基本デモ]({{ '/ja/samples/basic-demo' | relative_url }})、[Mote デモ]({{ '/ja/samples/mote-demo' | relative_url }})）と機能ガイドからお始めください。
{: .note }

---

## Tiny History とは？

<iframe width="560" height="315" src="https://www.youtube.com/embed/8it_Js-iabI" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

タイムライン操作機能付きの国家シミュレーションです。手続き生成されたマップ上で AI が操作する国家が領土を争い、タイムラインを巻き戻して別の歴史を探索できます。

- 国家は軍事征服により領土を拡大
- 軍隊が行進し、戦闘し、州を占領
- タイムラインスライダーで任意の時点にシーク可能
- イベントログが主要な歴史的出来事を記録

---

## デモで使用している機能

| 機能 | 用途 | 説明 |
| :--- | :--- | :--- |
| Event Channels | ドメインイベント | 州の所有権変更、国家滅亡 |
| Event Channels | UI通信 | タイムラインシークリクエスト |
| Reactive Variables | 状態同期 | 再生/一時停止、現在フレーム、シミュレーション速度 |
| Reactive Entity Sets | Jobs統合 | 軍隊、州、国家の状態（ダブルバッファリング） |
| カスタムGPUレンダリング | レンダリング | GraphicsBuffer経由での州の色、軍隊のインスタンシング |

---

## はじめに

### サンプルのインポート

1. **Window > Package Manager** を開く
2. リストから **Reactive SO** を選択
3. **Samples** セクションを展開
4. **Tiny History Demo** の横にある **Import** をクリック

### シーンを開く

インポートされたサンプルフォルダに移動し、`Scenes/TinyHistoryDemo.unity` を開きます。

### プレイ

Play Mode に入ると、国家が自動的に拡大と戦闘を開始します。

---

## UIコントロール

<!-- TODO: Add screenshot showing the UI controls (timeline slider, play/pause, speed slider, event log) -->

### タイムラインスライダー

ドラッグして記録された歴史の任意の時点にシークします。シーク時はシミュレーションが一時停止します。

### 再生 / 一時停止ボタン

シミュレーションの再生を切り替えます。一時停止中は、進行せずに歴史をシークできます。

### 速度スライダー

シミュレーション速度を 0.5x から 3x まで調整できます。速度を上げると歴史が速く進行します。

### イベントログ（年代記）

発生した歴史的イベントを表示します。

- **州の占領**: どの国家がどの領土を占領したか
- **国家滅亡**: 国家がすべての領土を失った時

「Key」（滅亡のみ）と「All」（全イベント）フィルターを切り替えられます。

### ステータスパネル

現在のシミュレーション状態を表示します。

- 現在の年
- 生存国家数
- アクティブな軍隊数
- 保存されたスナップショット数

---

## このガイドの内容

| ページ | 学べること |
| :--- | :--- |
| [アーキテクチャ](architecture) | システムとデータフローの視覚的な概要 |
| [Event Channels](event-channels) | このサンプルでの Event Channels の活用方法 |
| [Reactive Entity Sets](reactive-entity-sets) | このサンプルでの RES と Jobs・GPU の連携方法 |

---

## 主要ファイル

| ファイル | 説明 |
| :--- | :--- |
| `Scripts/TinyHistorySimulation.cs` | メインシミュレーションコントローラー |
| `Scripts/MapGenerator.cs` | 手続き的マップ生成 |
| `Scripts/MapRenderer.cs` | GPUベースの州レンダリング |
| `Scripts/ArmyRenderer.cs` | GPU インスタンス化による軍隊レンダリング |
| `Scripts/HistoryManager.cs` | スナップショットの取得と復元 |
| `Scenes/TinyHistoryDemo.unity` | メインデモシーン |

---

## さらに学ぶ

これらのパターンを自分のプロジェクトで使いたい場合は、機能ガイドをご覧ください。

| ガイド | 学べること |
| :--- | :--- |
| [Event Channels ガイド]({{ '/ja/guides/event-channels' | relative_url }}) | Event Channels の作成と使用方法 |
| [Variables ガイド]({{ '/ja/guides/variables' | relative_url }}) | GPU Sync を使用した Reactive Variables の使い方 |
| [Reactive Entity Sets ガイド]({{ '/ja/guides/reactive-entity-sets/' | relative_url }}) | Jobs とコレクションを使用した RES の詳細 |
