---
layout: default
title: トラブルシューティング
nav_order: 6
---

# トラブルシューティング

## 目的

このページでは、Reactive SOに関するよくある問題、既知の制限事項、よくある質問について説明します。

---

## よくある問題

### イベントが発火しない

**症状**: `RaiseEvent()`を呼び出しても何も起きない。

**チェックリスト**

1. **イベントチャンネルは割り当てられていますか？**
   - Inspectorで`None`になっていないか確認
   - 正しいScriptableObjectアセットをドラッグ

2. **null条件演算子を使用していますか？**
   ```csharp
   // Good
   eventChannel?.RaiseEvent();

   // Bad - 割り当てられていない場合NullReferenceExceptionが発生
   eventChannel.RaiseEvent();
   ```

3. **誰かが購読していますか？**
   - Event Monitorを開いて#L列を確認
   - Subscribers Listでリスナーを確認
   - 購読者のGameObjectがアクティブか確認

4. **イベントは実際に発火していますか？**
   - Event Monitorを開く
   - イベントが表示される場合、問題は購読者側にある
   - イベントが表示されない場合、発行者のコードを確認

### メモリリーク / イベントの重複

**症状**: イベントが複数回発火したり、シーン遷移後にメモリが増加する。

**原因**: `OnDisable`での購読解除を忘れている。

**修正**

```csharp
private void OnEnable()
{
    eventChannel.OnEventRaised += HandleEvent;
}

private void OnDisable()
{
    eventChannel.OnEventRaised -= HandleEvent;  // 必須!
}
```

**診断**: シーン遷移の前後でSubscribers Listを確認。破棄されたオブジェクトがリストに残っている場合、リークが発生しています。

### イベントが早すぎるタイミングで発火する

**症状**: イベント発火時に購読者の`Start()`がまだ実行されていない。

**解決策**:

**オプション1** - イベントを遅延させる
```csharp
private IEnumerator Start()
{
    yield return null;  // 1フレーム待つ
    onGameStarted?.RaiseEvent();
}
```

**オプション2** - より早く購読する
```csharp
private void Awake()
{
    eventChannel.OnEventRaised += HandleEvent;
}

private void OnDisable()
{
    eventChannel.OnEventRaised -= HandleEvent;
}
```

### 間違ったイベントチャンネルを参照している

**症状**: 複数のイベントチャンネルが似た名前を持っており、間違ったものを参照している。

**修正** 明確な名前を使用し、フォルダで整理しましょう。

```text
ScriptableObjects/Events/
├── Player/
│   ├── OnPlayerDeath.asset
│   └── OnPlayerHealthChanged.asset
├── UI/
│   └── OnMenuOpened.asset
└── Game/
    └── OnGamePaused.asset
```

### 購読者が更新されない

**症状**: イベントは発火するが購読者が反応しない。

**チェックリスト**

1. GameObjectはアクティブですか？
2. MonoBehaviourは有効ですか？
3. `OnEnable()`は実行されましたか？
4. 購読は正しいですか？
   ```csharp
   // Correct
   eventChannel.OnEventRaised += HandleEvent;

   // Wrong - 他の購読者を上書きする
   eventChannel.OnEventRaised = HandleEvent;
   ```
5. メソッドシグネチャは一致していますか？
   ```csharp
   // IntEventChannelSOの場合
   void HandleEvent(int value) { }  // Correct
   void HandleEvent() { }           // Wrong signature
   ```

---

## 既知の制限事項

### UnityEventsでの呼び出し元情報

呼び出し元情報は、直接コードから呼び出す場合にのみ機能します。UnityEvent経由でイベントを発生させた場合（例：InspectorのButton.OnClick）は機能しません。

**機能する**
```csharp
public void OnButtonClick()
{
    onButtonPressed?.RaiseEvent();  // Caller: "OnButtonClick"
}
```

**機能しない**
```text
Inspector: Button.OnClick → EventChannel.RaiseEvent
Caller shows: "-"
```

**回避策**: Inspectorから`RaiseEvent`を直接呼び出すのではなく、ラッパーメソッドを作成してください。

### Manual Triggerのカスタム型

Manual Triggerは12種類の組み込みイベント型のみをサポートしています。カスタム型の場合、トリガーボタンの代わりにヘルプメッセージが表示されます。

### Subscribers Listのスコープ

Subscribers ListにはMonoBehaviourの購読者のみが表示されます。その他の購読者（ScriptableObject、静的クラス）は#Lカウントには含まれますが、リストには表示されません。

---

## FAQ

### Reactive SOはビルドで使用できますか？

はい。すべての機能がビルドで動作し、オーバーヘッドはゼロです。デバッグツール（Event Monitor、Manual Trigger、Subscribers List、Caller Info）はEditorのみで動作し、ビルドから自動的に削除されます。

### パフォーマンスコストは？

ほぼゼロです。イベントチャンネルは内部的に標準的なC#イベントを使用しています。デバッグ機能はビルドから完全に削除されます。

### イベントチャンネルをコルーチンと一緒に使用できますか？

はい、使用できます。

```csharp
private void OnEnable()
{
    eventChannel.OnEventRaised += HandleEvent;
}

private void HandleEvent()
{
    StartCoroutine(HandleEventCoroutine());
}

private IEnumerator HandleEventCoroutine()
{
    // Your coroutine logic
}
```

### 1つのイベントで複数の値を渡せますか？

直接的には不可能です。以下のアプローチのいずれかを使用してください:

**オプション1** - 構造体を使用する
```csharp
[System.Serializable]
public struct PlayerDeathData
{
    public Vector3 position;
    public string killerName;
    public int score;
}

public class PlayerDeathEventChannelSO : EventChannelSO<PlayerDeathData> { }
```

**オプション2** - 複数のイベントを使用する
```csharp
onPlayerDeath?.RaiseEvent();
onDeathPosition?.RaiseEvent(position);
onFinalScore?.RaiseEvent(score);
```

### ScriptableObjectから購読できますか？

はい、購読できます。

```csharp
public class GameSettings : ScriptableObject
{
    [SerializeField] private VoidEventChannelSO onGameStarted;

    private void OnEnable()
    {
        onGameStarted.OnEventRaised += HandleGameStarted;
    }

    private void OnDisable()
    {
        onGameStarted.OnEventRaised -= HandleGameStarted;
    }

    private void HandleGameStarted()
    {
        // Initialize settings
    }
}
```

注意: ScriptableObjectの購読者はSubscribers Listには表示されませんが、#L列にはカウントされます。

### イベントチャンネルのユニットテストはどうすればいいですか？

```csharp
using NUnit.Framework;
using Tang3cko.ReactiveSO;

public class EventChannelTests
{
    [Test]
    public void EventRaisesCorrectly()
    {
        // Arrange
        var eventChannel = ScriptableObject.CreateInstance<IntEventChannelSO>();
        int receivedValue = 0;
        eventChannel.OnEventRaised += (value) => receivedValue = value;

        // Act
        eventChannel.RaiseEvent(42);

        // Assert
        Assert.AreEqual(42, receivedValue);
    }
}
```

### イベントチャンネルをネットワーキングに使用できますか？

イベントチャンネルは単一のUnityインスタンス内でローカルに動作します。ネットワークゲームの場合は、以下のアプローチを検討してください。

- ローカルゲームロジックにはイベントチャンネルを使用
- ネットワーク通信にはネットワーキングソリューションを使用
- ネットワークイベントに応答してイベントチャンネルを発火

以下は、ネットワークコールバックからローカルイベントを発火する例です。

```csharp
[ClientRpc]
void RpcPlayerDeath()
{
    // すべてのクライアントで実行
    onPlayerDeath?.RaiseEvent();  // UI/オーディオ用のローカルイベント
}
```

---

## まだ問題がありますか？

ここで説明されていない問題が発生している場合は、以下の手順をお試しください。

1. Event Monitorを確認してイベントフローを検証
2. Subscribers Listを使用して購読を検証
3. Manual Triggerを試して発行者と購読者の問題を切り分け
4. `OnEnable`、`OnDisable`、ハンドラーに`Debug.Log`を追加
5. 基本的なセットアップについては[Getting Started]({{ '/ja/getting-started' | relative_url }})を確認

---

## 問題の報告

問題を報告する際は、以下の情報を含めてください。

- Unityバージョン
- Reactive SOバージョン
- 再現手順
- Event Monitorの出力
- Subscribers Listのスクリーンショット

サポートについては、以下のチャンネルをご利用ください。

- [Report issues on GitHub](https://github.com/tang3cko/ReactiveSO-docs/issues)
- [Asset Store page](https://assetstore.unity.com/packages/tools/game-toolkits/reactive-so-339926)
- [X (@tang3cko)](https://x.com/tang3cko)

---

## 参考資料

- [Event Channels Guide]({{ '/ja/guides/event-channels' | relative_url }}) - 基本的な使用方法
- [Debugging]({{ '/ja/debugging/' | relative_url }}) - デバッグツール
- [Event Monitor]({{ '/ja/debugging/event-monitor' | relative_url }}) - リアルタイムトラッキング
