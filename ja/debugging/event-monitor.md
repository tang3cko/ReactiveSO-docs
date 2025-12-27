---
layout: default
title: Event monitor
parent: デバッグ
nav_order: 2
---

# Event monitor

## 目的

このガイドでは、リアルタイムイベント追跡のためのEvent Monitor Windowの使用方法を説明します。イベントフローの問題の特定、重複イベントの発見、サブスクライバー数の確認方法を学びます。

---

## ウィンドウを開く

**Window > Reactive SO > Event Monitor**に移動します。

<!-- TODO: Add screenshot of Event Monitor Window showing event log with Time, Event Name, Type, Value, #L, and Caller columns -->

---

## イベントのフィルタリング

**Show In Event Log**が有効になっているイベントチャンネルのみがこのウィンドウに表示されます。新しいイベントチャンネルではデフォルトで有効になっています。

イベントのログを無効にするには:

1. イベントチャンネルアセットを選択
2. Inspectorでデバッグセクションを見つける
3. "Show In Event Log"を無効化

これにより、デバッグしていない高頻度イベントのノイズを減らすことができます。

---

## ウィンドウの列

| 列 | 説明 |
|--------|-------------|
| Time | Play Mode開始からの秒数 |
| Event Name | イベントチャンネルアセットの名前 |
| Type | イベントの型 (Void、Int、Floatなど) |
| Value | イベントと一緒に渡された値 |
| #L | アクティブなリスナーの数 |
| Caller | イベントを発生させたコードの場所 |

---

## コントロール

| コントロール | 説明 |
|---------|-------------|
| Search | 名前または型でイベントを絞り込み |
| Pause/Resume | ログを一時停止してイベントを調査 |
| Auto-scroll | 新しいイベントに自動スクロール |
| Export CSV | 外部分析用にログを保存 |
| Clear | ログに記録されたすべてのイベントを削除 |
| Max | 最大ログエントリー数 (100-10K) |

---

## 呼び出し元情報の理解

Caller列は、各イベントを発生させたコードを表示します:

```text
FileName.cs:MethodName:LineNumber
```

例:

```text
PlayerController.cs:TakeDamage:142
```

これは以下を示します:

- **File**: PlayerController.cs
- **Method**: TakeDamage()
- **Line**: 142

(サポートされている場合)呼び出し元をクリックすると、IDEでファイルが開きます。

### 仕組み

呼び出し元情報はコンパイル時にC#の`CallerInfo`属性を使用します。これは、ランタイムオーバーヘッドがゼロであることを意味します。

### 制限事項: UnityEvents

UnityEvent経由でイベントが発生した場合(例: InspectorのButton.OnClick)、呼び出し元情報は機能しません。Caller列には"-"が表示されます。

**回避策**: ラッパーメソッドを作成します:

```csharp
// Inspectorで: Button.OnClick -> OnButtonClick
public void OnButtonClick()
{
    onButtonPressed?.RaiseEvent();  // Caller: "OnButtonClick"
}
```

---

## ユースケース

### このイベントは発火していますか?

ボタンが機能しない場合:

1. Event Monitorを開く
2. ゲーム内でボタンをクリック
3. イベントが表示されるかチェック

**イベントが表示される**: 問題はサブスクライバー側。
**イベントが表示されない**: パブリッシャーと`RaiseEvent()`呼び出しを確認。

### 重複イベントの発見

サウンドが2回再生される場合:

1. Event Monitorを開く
2. プレイヤーの死亡を1回トリガー
3. ログのエントリー数をカウント
4. Caller列を確認:

```text
Time | Event Name    | Caller
0.5s | OnPlayerDeath | EnemyAI.cs:Attack:142
0.5s | OnPlayerDeath | FallDetector.cs:OnTriggerEnter:28
```

2つのスクリプトが同時に同じイベントを発生させました。

### イベントチェーンの追跡

OnPlayerDeath -> OnGameOver -> OnHighScoreSavedのような複雑なチェーン:

1. Event Monitorを開く
2. プレイヤーの死亡をトリガー
3. 完全なシーケンスを確認:

```text
Time | Event Name       | Caller
0.5s | OnPlayerDeath    | Player.cs:TakeDamage:89
0.6s | OnGameOver       | GameManager.cs:HandlePlayerDeath:45
0.7s | OnHighScoreSaved | HighScoreManager.cs:HandleGameOver:67
```

### リスナー数の確認

3つのシステムが応答すべきなのに2つしか応答しない場合:

1. Event Monitorを開く
2. イベントをトリガー
3. #L列をチェック
4. #Lが2を示している場合、1つのシステムがサブスクライブを忘れています

Subscribers Listを使用してどれかを見つけます。

---

## 検索とフィルタ

検索ボックスを使用してイベントを絞り込み:

- イベント名: `OnPlayerDeath`
- イベント型: `Void`、`Int`、`Float`
- 部分一致: `Player`は`OnPlayerDeath`、`OnPlayerJumped`にマッチ

例:

- `Death`を入力して死亡関連のイベントを表示
- `Int`を入力して整数イベントを表示
- `UI`を入力してUI関連のイベントを表示

---

## 一時停止と分析

**Pause**をクリックしてログを停止:

- イベントが速すぎて読めない場合
- 複数のイベントエントリーを比較する場合
- 特定のシーケンスを調査する場合

**Resume**をクリックしてログ記録を再開します。

---

## ログの永続性

イベントログは以下のように動作します:

| アクション | 動作 |
|--------|----------|
| Play Modeに入る | ログがクリアされる |
| Play Mode中 | イベントが記録される |
| Play Modeを抜ける | レビュー用にログが保持される |
| 次のPlay Mode | 前のログがクリアされる |

Play Modeを抜けた後もログを確認できます。

---

## ログのエクスポート

**Export CSV**をクリックしてログを保存します。ファイルには以下が含まれます:

- Time、Event Name、Type、Value、Listeners、Caller

CSVファイルはタイムスタンプ付きの名前で、Excel互換性のためにUTF-8エンコーディングで保存されます。

エクスポートの用途:

- イベント頻度分析
- パフォーマンスプロファイリング
- メモリリーク検出
- プレイテストデータ分析

---

## 設定

### 最大ログエントリー数

1. ツールバーのMaxドロップダウンを見つける
2. 100、200、500、1K、2K、5K、10Kから選択
3. 設定は自動的に保存されます

高い値を使用する場合:

- 長いイベントシーケンス
- 拡張デバッグセッション
- パターン分析

注意: 高い値はより多くのメモリを消費します。

---

## パフォーマンス

Event Monitorのパフォーマンスへの影響は無視できる程度です:

- ログ記録はエディターでのみ行われます(ビルドでは行われません)
- 効率的な文字列フォーマット
- 自動エントリー制限
- 必要ないときは閉じることができます

---

## ヒント

- セカンドモニターにウィンドウを開いたままにする
- 検索を使用して特定のイベントに焦点を当てる
- 複雑なシーケンスの前に一時停止する
- 呼び出し元をクリックしてソースコードにジャンプ
- 高頻度イベントのログを無効化する

---

## 参考資料

- [デバッグ概要]({{ '/ja/debugging/' | relative_url }}) - すべてのデバッグツール
- [Dependency Analyzer](dependency-analyzer) - 静的解析
- [トラブルシューティング]({{ '/ja/troubleshooting' | relative_url }}) - 一般的な問題
