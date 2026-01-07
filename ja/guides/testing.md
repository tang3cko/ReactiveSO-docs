---
layout: default
title: Testing
parent: ガイド
nav_order: 6
---

# テスト

{: .note }
> このガイドでは、Unity Test Framework（NUnit）を使用したReactive SOコンポーネントのユニットテストパターンを解説します。

---

## 目的

このガイドでは、Reactive SOコンポーネントのユニットテストの書き方を説明します。Event Channels、Variables、Runtime Sets、Reactive Entity Setsのテストパターンと、外部依存を分離するための依存性注入テクニックを学びます。

---

## 前提条件

### Unity Test Framework

Unity Test FrameworkはUnityに同梱されています。Test Runnerは以下のメニューからアクセスします。

```text
Window > General > Test Runner
```

### Edit Mode vs Play Mode

| モード | 用途 | 速度 |
|--------|------|------|
| **Edit Mode** | ScriptableObjectロジック、純粋なC#クラス | 高速 |
| **Play Mode** | MonoBehaviourライフサイクル、シーン統合 | 低速 |

ScriptableObjectはシーンなしでインスタンス化できるため、ほとんどのReactive SOテストはEdit Modeで実行します。

---

## Event Channelsのテスト

### 基本パターン

テストでScriptableObjectインスタンスを直接作成します。

```csharp
using NUnit.Framework;
using UnityEngine;
using Tang3cko.ReactiveSO;

public class IntEventChannelSOTests
{
    private IntEventChannelSO channel;

    [SetUp]
    public void Setup()
    {
        // アセットファイルなしでインスタンスを作成
        channel = ScriptableObject.CreateInstance<IntEventChannelSO>();
    }

    [TearDown]
    public void Teardown()
    {
        // Edit ModeではDestroyImmediateを使用
        Object.DestroyImmediate(channel);
    }

    [Test]
    public void RaiseEvent_WithSubscriber_NotifiesSubscriberWithValue()
    {
        // Arrange
        int receivedValue = 0;
        channel.OnEventRaised += (value) => receivedValue = value;

        // Act
        channel.RaiseEvent(42);

        // Assert
        Assert.That(receivedValue, Is.EqualTo(42));
    }
}
```

### 複数サブスクライバーのテスト

```csharp
[Test]
public void RaiseEvent_WithMultipleSubscribers_NotifiesAll()
{
    // Arrange
    int notificationCount = 0;
    channel.OnEventRaised += (value) => notificationCount++;
    channel.OnEventRaised += (value) => notificationCount++;
    channel.OnEventRaised += (value) => notificationCount++;

    // Act
    channel.RaiseEvent(10);

    // Assert
    Assert.That(notificationCount, Is.EqualTo(3));
}
```

### 購読解除のテスト

```csharp
[Test]
public void Unsubscribe_AfterRaiseEvent_DoesNotNotify()
{
    // Arrange
    bool wasNotified = false;
    System.Action<int> handler = (value) => wasNotified = true;
    channel.OnEventRaised += handler;
    channel.OnEventRaised -= handler;

    // Act
    channel.RaiseEvent(42);

    // Assert
    Assert.That(wasNotified, Is.False);
}
```

---

## Variablesのテスト

### 基本パターン

```csharp
public class IntVariableSOTests
{
    private IntVariableSO variable;
    private IntEventChannelSO eventChannel;

    [SetUp]
    public void Setup()
    {
        variable = ScriptableObject.CreateInstance<IntVariableSO>();
        eventChannel = ScriptableObject.CreateInstance<IntEventChannelSO>();
    }

    [TearDown]
    public void Teardown()
    {
        Object.DestroyImmediate(variable);
        Object.DestroyImmediate(eventChannel);
    }

    [Test]
    public void Value_Set_ChangesValue()
    {
        // Arrange
        variable.Value = 10;

        // Act
        variable.Value = 20;

        // Assert
        Assert.That(variable.Value, Is.EqualTo(20));
    }
}
```

### 変更検出のテスト

Variablesは値が実際に変更されたときのみイベントを発火します。

```csharp
[Test]
public void Value_SetSameValue_DoesNotRaiseEvent()
{
    // Arrange
    variable.Value = 42;
    bool eventRaised = false;

    // リフレクションでイベントチャネルを設定（内部フィールド）
    var field = typeof(VariableSO<int>).GetField("onValueChanged",
        System.Reflection.BindingFlags.NonPublic |
        System.Reflection.BindingFlags.Instance);
    field.SetValue(variable, eventChannel);

    eventChannel.OnEventRaised += (value) => eventRaised = true;

    // Act
    variable.Value = 42;  // 同じ値

    // Assert
    Assert.That(eventRaised, Is.False);
}
```

---

## Runtime Setsのテスト

### 基本パターン

```csharp
public class GameObjectRuntimeSetSOTests
{
    private GameObjectRuntimeSetSO runtimeSet;
    private GameObject testObject;

    [SetUp]
    public void Setup()
    {
        runtimeSet = ScriptableObject.CreateInstance<GameObjectRuntimeSetSO>();
        testObject = new GameObject("TestObject");
    }

    [TearDown]
    public void Teardown()
    {
        Object.DestroyImmediate(testObject);
        Object.DestroyImmediate(runtimeSet);
    }

    [Test]
    public void Add_ValidObject_IncreasesCount()
    {
        // Arrange
        Assert.That(runtimeSet.Count, Is.EqualTo(0));

        // Act
        runtimeSet.Add(testObject);

        // Assert
        Assert.That(runtimeSet.Count, Is.EqualTo(1));
    }
}
```

---

## Reactive Entity Setsのテスト

### テスト専用サブクラスパターン

ジェネリッククラスをテストするために具象実装を作成します。

```csharp
public class ReactiveEntitySetSOTests
{
    // テストデータ構造
    public struct TestEntityData
    {
        public int Health;
        public float Speed;
    }

    // テスト実装
    private class TestReactiveEntitySetSO : ReactiveEntitySetSO<TestEntityData>
    {
    }

    // テスト用オーナーコンポーネント
    private class TestOwner : MonoBehaviour { }

    private TestReactiveEntitySetSO entitySet;
    private GameObject ownerObject;
    private TestOwner owner;

    [SetUp]
    public void Setup()
    {
        entitySet = ScriptableObject.CreateInstance<TestReactiveEntitySetSO>();
        ownerObject = new GameObject("Owner");
        owner = ownerObject.AddComponent<TestOwner>();
    }

    [TearDown]
    public void Teardown()
    {
        Object.DestroyImmediate(ownerObject);
        Object.DestroyImmediate(entitySet);
    }

    [Test]
    public void Register_ValidOwner_AddsToSet()
    {
        // Arrange
        var data = new TestEntityData { Health = 100, Speed = 5f };

        // Act
        entitySet.Register(owner, data);

        // Assert
        Assert.That(entitySet.Count, Is.EqualTo(1));
        Assert.That(entitySet.Contains(owner), Is.True);
    }

    [Test]
    public void UpdateData_ModifiesState()
    {
        // Arrange
        entitySet.Register(owner, new TestEntityData { Health = 100 });

        // Act
        entitySet.UpdateData(owner, state => {
            state.Health -= 30;
            return state;
        });

        // Assert
        var data = entitySet.GetData(owner);
        Assert.That(data.Health, Is.EqualTo(70));
    }
}
```

---

## 依存性注入パターン

外部依存（ファイルI/O、ダイアログ）を持つクラスには、インターフェースと手動モックを使用します。

### インターフェースの定義

```csharp
public interface IFileService
{
    string SaveFilePanel(string title, string directory,
        string defaultName, string extension);
    void WriteAllText(string path, string content, Encoding encoding);
    void RevealInFinder(string path);
}
```

### モック実装の作成

```csharp
public class MockFileService : IFileService
{
    public string PathToReturn { get; set; }
    public bool WriteAllTextCalled { get; private set; }
    public string LastWrittenPath { get; private set; }
    public string LastWrittenContent { get; private set; }

    public string SaveFilePanel(string title, string directory,
        string defaultName, string extension)
    {
        return PathToReturn;
    }

    public void WriteAllText(string path, string content, Encoding encoding)
    {
        WriteAllTextCalled = true;
        LastWrittenPath = path;
        LastWrittenContent = content;
    }

    public void RevealInFinder(string path) { }
}
```

### テストでの使用

```csharp
public class ExporterTests
{
    private MockFileService mockFileService;
    private MockDialogService mockDialogService;
    private Exporter exporter;

    [SetUp]
    public void Setup()
    {
        mockFileService = new MockFileService();
        mockDialogService = new MockDialogService();
        exporter = new Exporter(mockDialogService, mockFileService);
    }

    [Test]
    public void Export_WithValidPath_WritesFile()
    {
        // Arrange
        mockFileService.PathToReturn = "/path/to/file.csv";

        // Act
        exporter.Export(testData);

        // Assert
        Assert.That(mockFileService.WriteAllTextCalled, Is.True);
        Assert.That(mockFileService.LastWrittenPath,
            Is.EqualTo("/path/to/file.csv"));
    }

    [Test]
    public void Export_UserCancels_DoesNotWriteFile()
    {
        // Arrange
        mockFileService.PathToReturn = "";  // ユーザーがキャンセル

        // Act
        exporter.Export(testData);

        // Assert
        Assert.That(mockFileService.WriteAllTextCalled, Is.False);
    }
}
```

---

## ベストプラクティス

### AAAパターンに従う

テストを明確なセクションで構造化します。

```csharp
[Test]
public void MethodName_Scenario_ExpectedBehavior()
{
    // Arrange - テストデータと条件をセットアップ

    // Act - テスト対象のコードを実行

    // Assert - 結果を検証
}
```

### 常にクリーンアップ

Edit ModeテストではDestroyImmediateを使用します。

```csharp
[TearDown]
public void Teardown()
{
    // DestroyはEdit Modeでは即座に機能しない
    Object.DestroyImmediate(channel);
}
```

### エッジケースをテスト

```csharp
[Test]
public void RaiseEvent_WithoutSubscribers_DoesNotThrow()
{
    Assert.DoesNotThrow(() => channel.RaiseEvent(42));
}

[Test]
public void RaiseEvent_WithMinValue_TransmitsMinValue()
{
    int receivedValue = 0;
    channel.OnEventRaised += (value) => receivedValue = value;

    channel.RaiseEvent(int.MinValue);

    Assert.That(receivedValue, Is.EqualTo(int.MinValue));
}
```

### テストの独立性を保つ

各テストは単独で実行可能であるべきです。

```csharp
// 悪い例：テスト間で状態を共有
private static IntEventChannelSO sharedChannel;

// 良い例：各テストが独自のインスタンスを作成
[SetUp]
public void Setup()
{
    channel = ScriptableObject.CreateInstance<IntEventChannelSO>();
}
```

---

## よくある落とし穴

### ScriptableObjectの破棄忘れ

`CreateInstance`で作成したScriptableObjectは破棄するまで残り続けます。常に`TearDown`でクリーンアップしてください。

### `CreateInstance`の代わりに`new`を使用

```csharp
// 悪い例：エラーになる
var channel = new IntEventChannelSO();

// 良い例：ScriptableObjectの正しい作成方法
var channel = ScriptableObject.CreateInstance<IntEventChannelSO>();
```

### 不必要にPlay Modeでテスト

ほとんどのReactive SOロジックはEdit Modeでテスト可能であり、より高速です。

---

## まとめ

| コンポーネント | 主要パターン |
|---------------|-------------|
| Event Channels | CreateInstance、サブスクライブ、コールバック検証 |
| Variables | CreateInstance、値設定、変更検出確認 |
| Runtime Sets | GameObject作成、追加/削除、カウント検証 |
| Reactive Entity Sets | テストサブクラス、Register/UpdateData、状態検証 |
| 外部依存 | インターフェース + モックパターン |

---

## 参照

- [テスト容易性]({{ '/ja/design-philosophy/testability' | relative_url }}) - Reactive SOがなぜテスト可能か
- [Unity Test Framework](https://docs.unity3d.com/Packages/com.unity.test-framework@latest) - 公式ドキュメント
- [NUnit Documentation](https://docs.nunit.org/) - テストフレームワークリファレンス
