---
layout: default
title: デバッグ
nav_order: 6
has_children: true
---

# デバッグ

---

## 目的

このセクションでは、Reactive SOのイベントフローをデバッグし、問題を診断する方法を説明します。組み込みのデバッグツールの使い方と、一般的な問題のトラブルシューティング方法を学びます。

---

## 利用可能なツール

Reactive SOは以下のデバッグツールを提供しています:

| ツール | 目的 |
|------|---------|
| [Event Monitor](event-monitor) | 呼び出し元情報を含むリアルタイムイベント追跡 |
| [Variable Monitor](variable-monitor) | リアルタイム変数状態監視 |
| [Runtime Set Monitor](runtime-set-monitor) | リアルタイムランタイムセット監視 |
| [Dependency Analyzer](dependency-analyzer) | シーンとプレハブの静的解析 |
| Manual Trigger | コードなしでInspectorからイベントをテスト |
| Subscribers List | Play Mode中のアクティブなサブスクライバーを表示 |

---

## ツール選択ガイド

| 質問 | ツール |
|----------|------|
| イベントは発火していますか? | Event Monitor |
| 現在の変数の値は何ですか? | Variable Monitor |
| ランタイムセットにどのオブジェクトが含まれていますか? | Runtime Set Monitor |
| このイベントには誰がサブスクライブしていますか? | Subscribers List |
| このイベントチャンネルはどこで使用されていますか? | Dependency Analyzer |
| プレイせずにテストできますか? | Manual Trigger |
| このイベントを発生させたコードはどれですか? | Event Monitor (Caller列) |
| メモリリークはありますか? | Subscribers List (シーン遷移時) |
| すべてのイベントチャンネルは割り当てられていますか? | Dependency Analyzer |
| イベントと一緒に渡された値は何ですか? | Event Monitor (Value列) |

---

## 一般的なデバッグシナリオ

### イベントが発火しない

**問題**: ボタンクリックが期待される動作をトリガーしません。

**手順**:

1. Event Monitorを開く (Window > Reactive SO > Event Monitor)
2. イベントチャンネルで"Show In Event Log"が有効になっていることを確認
3. ゲーム内でボタンをクリック
4. イベントがログに表示されるかチェック

**診断**:

- イベントが表示される: 問題はサブスクライバー側。Subscribers Listを確認。
- イベントが表示されない: 問題はパブリッシャー側。イベントチャンネルの割り当てと`RaiseEvent()`呼び出しを確認。

### イベントは発火するが何も起こらない

**問題**: イベントがEvent Monitorに表示されるが期待される動作が発生しません。

**手順**:

1. Event Monitorの#L列をチェック (リスナー数)
2. #L = 0の場合、誰もサブスクライブしていません
3. イベントチャンネルを選択してSubscribers Listを確認
4. 期待されるMonoBehaviourがリストされているか確認

**診断**:

- サブスクライバーが存在する: サブスクライバーメソッドのロジックを確認
- サブスクライバーが存在しない: `OnEnable`/`OnDisable`のサブスクリプションコードを確認
- 間違ったサブスクライバー: イベントチャンネルの割り当てを確認

### イベントが複数回発火する

**問題**: サウンドが2回再生されるか、UIが複数回更新されます。

**手順**:

1. Event Monitorを開く
2. イベントを1回トリガー
3. ログのエントリー数をカウント
4. Caller列で重複するソースをチェック

**診断**:

```text
Time | Event Name    | Caller
0.5s | OnPlayerDeath | EnemyAI.cs:Attack:142
0.5s | OnPlayerDeath | FallDetector.cs:OnTriggerEnter:28
```

2つのスクリプトが同じイベントを発生させています。ゲームロジックを確認してください。

### シーン遷移後のメモリリーク

**問題**: 複数回のシーンロード後にメモリ使用量が増加します。

**手順**:

1. Scene 1でPlay Modeに入る
2. イベントチャンネルのSubscribers Listを確認
3. Scene 2をロード
4. Scene 1に戻る
5. 再度Subscribers Listを確認

**診断**:

```text
期待される状態:
  UIManager.HandleEvent
  PlayerController.HandleEvent

メモリリーク:
  UIManager.HandleEvent (destroyed)  <- リーク
  PlayerController.HandleEvent (destroyed)  <- リーク
  UIManager.HandleEvent
  PlayerController.HandleEvent
```

**修正**: `OnDisable`でサブスクリプション解除を追加:

```csharp
private void OnDisable()
{
    eventChannel.OnEventRaised -= HandleEvent;
}
```

### リファクタリング前にすべての使用箇所を見つける

**問題**: イベントチャンネルの名前を変更または削除する必要がありますが、どこで使用されているかわかりません。

**手順**:

1. Dependency Analyzerを開く (Window > Reactive SO > Dependency Analyzer)
2. "Scan Project"をクリック
3. イベントチャンネル名を検索
4. 展開してすべての使用箇所を表示

**結果**:

```text
OnPlayerDeath (VoidEventChannelSO) - 5 usage(s)
  MainScene.unity > Player.PlayerController
  MainScene.unity > UICanvas.UIManager
  BossScene.unity > Boss.BossAI
  PlayerPrefab.prefab > Player.HealthComponent
  GameOverUI.prefab > GameOverPanel.GameOverController
```

### 未割り当てフィールドの検出

**問題**: 未割り当てのイベントチャンネルによるランタイムNullReferenceException。

**手順**:

1. Dependency Analyzerを開く
2. "Scan Project"をクリック
3. Summary行で未割り当てフィールド数を確認
4. 未割り当てフィールドセクションを確認

**結果**:

```text
Summary: 20 Event Channels, 1 unused, 3 unassigned fields

Unassigned Event Channel Fields:
  MainScene.unity > Player.PlayerController.onPlayerDeath
  EnemyPrefab.prefab > Enemy.EnemyAI.onSpawn
```

クリックして各場所にジャンプして修正します。

---

## デバッグ設定

各イベントチャンネルのInspectorにはデバッグオプションがあります:

### Show In Event Log

有効にすると、Play Mode中にイベントがEvent Monitorに表示されます。新しいチャンネルではデフォルトで有効です。

### Show In Console

有効にすると、イベントがUnity Consoleに出力されます。デフォルトでは無効です。従来のDebug.Logスタイルのデバッグに使用します。

<!-- TODO: Add screenshot of Event Channel Inspector showing Debug settings section -->

---

## 新機能開発のワークフロー

### 開発

1. イベントチャンネルを作成
2. "Show In Event Log"を有効化
3. `RaiseEvent()`でパブリッシャーを実装
4. Manual Triggerでテスト
5. サブスクライバーを実装
6. Subscribers Listで検証

### テスト

1. Event Monitorを開いたままにする
2. 機能を実行
3. 正しいタイミングでイベントが発火することを確認
4. Subscribers Listでメモリリークを確認

### リリース前

1. Dependency Analyzerを実行
2. 未割り当てフィールドを修正
3. 本番環境用に"Show In Event Log"を無効化

---

## 問題が発生しましたか?

一般的な問題と解決策については、[トラブルシューティング]({{ '/ja/troubleshooting' | relative_url }})を参照してください。
