---
layout: default
title: Action Types
parent: リファレンス
nav_order: 5
---

# Action Types

{: .note }
> Actionsはv2.1.0から利用可能です。

## 目的

このリファレンスでは、データ駆動で再利用可能なコマンドを作成するためのAction APIを文書化しています。ActionSO基底クラスのAPIリファレンス、実装パターン、例を確認できます。

---

## タイプ概要

| タイプ | 説明 |
|--------|------|
| ActionSO | パラメータなしアクションの基底クラス |
| ActionSO&lt;T&gt; | パラメータ付きアクションのジェネリック基底クラス |

---

## ActionSO

Commandパターンを実装するScriptableObjectベースのアクション用抽象基底クラス。

### プロパティ

| プロパティ | 型 | 説明 |
|------------|------|------|
| Description | `string` | Inspectorに表示されるユーザー定義の説明 |

### メソッド

| メソッド | 戻り値 | 説明 |
|----------|--------|------|
| `Execute(...)` | `void` | アクションを実行（非virtual。呼び出し元情報の記録とMonitor通知を一元管理し、内部で`OnExecute()`を呼び出す） |
| `OnExecute(...)` | `void` | アクションの実際の処理を実装（抽象、オーバーライド必須） |

### エディタ専用プロパティ

| プロパティ | 型 | 説明 |
|------------|------|------|
| showInMonitor | `bool` | Play Mode中にMonitor Windowに表示 |
| showInConsole | `bool` | 実行をConsoleにログ出力 |

### エディタ専用メソッド

| メソッド | 戻り値 | 説明 |
|----------|--------|------|
| `LogAction(string)` | `void` | アクション実行をConsoleにログ出力（`OnExecute()`内から呼び出す） |

### エディタ専用イベント

| イベント | 型 | 説明 |
|----------|------|------|
| OnAnyActionExecuted | `Action<ActionSO, CallerInfo>` | 監視用静的イベント |

---

## ActionSO&lt;T&gt;

実行時にパラメータを受け取るアクション用ジェネリック基底クラス。

### メソッド

| メソッド | 戻り値 | 説明 |
|----------|--------|------|
| `Execute(T value, ...)` | `void` | パラメータ付きで実行（非virtual。内部で`OnExecute(T value)`を呼び出す） |
| `OnExecute(T value)` | `void` | パラメータ付きの実際の処理を実装（抽象、オーバーライド必須） |
| `Execute(...)` | `void` | デフォルト値で実行（内部で`OnExecute(default)`を呼び出す） |

---

## ActionSOの実装

パラメータなしの基本的な実装。

### 作成

```csharp
using Tang3cko.ReactiveSO;
using UnityEngine;

[CreateAssetMenu(
    fileName = "PlaySound",
    menuName = "Game/Actions/Play Sound"
)]
public class PlaySoundAction : ActionSO
{
    [Header("Settings")]
    [SerializeField] private AudioClip clip;
    [SerializeField] private float volume = 1f;

    protected override void OnExecute()
    {
        AudioSource.PlayClipAtPoint(clip, Vector3.zero, volume);

#if UNITY_EDITOR
        LogAction($"Played {clip.name}");
#endif
    }
}
```

### 使用方法

```csharp
[SerializeField] private ActionSO playSoundAction;

public void OnButtonClick()
{
    playSoundAction?.Execute();
}
```

---

## ActionSO&lt;T&gt;の実装

パラメータ付きの実装。

### 作成

```csharp
using Tang3cko.ReactiveSO;
using UnityEngine;

[CreateAssetMenu(
    fileName = "SpawnAtPosition",
    menuName = "Game/Actions/Spawn At Position"
)]
public class SpawnAtPositionAction : ActionSO<Vector3>
{
    [Header("Settings")]
    [SerializeField] private GameObject prefab;

    protected override void OnExecute(Vector3 position)
    {
        Object.Instantiate(prefab, position, Quaternion.identity);

#if UNITY_EDITOR
        LogAction($"Spawned at {position}");
#endif
    }
}
```

### 使用方法

```csharp
[SerializeField] private ActionSO<Vector3> spawnAction;

public void SpawnEnemy()
{
    Vector3 spawnPoint = GetRandomSpawnPoint();
    spawnAction?.Execute(spawnPoint);
}
```

---

## 呼び出し元情報

アクションはデバッグ用に呼び出し元情報を自動追跡します。

### CallerInfo構造体

| フィールド | 型 | 説明 |
|------------|------|------|
| MemberName | `string` | 呼び出しメソッド名 |
| FilePath | `string` | 呼び出しファイルのフルパス |
| LineNumber | `int` | 呼び出し行番号 |

### フォーマット済み出力

```csharp
// CallerInfo.ToString() の戻り値:
// "FileName.cs:MethodName:42"
```

### 重要な注意点

- callerパラメータに明示的な値を渡さない
- コンパイラに自動的に埋めさせる
- デフォルトのままにした場合のみ値が意味を持つ

```csharp
// 良い例：コンパイラに呼び出し元情報を埋めさせる
action.Execute();

// 悪い例：明示的な値は呼び出し元追跡を失う
action.Execute("", "", 0);
```

---

## Monitor統合

アクションはリアルタイムデバッグのためにMonitor Windowと統合されています。

### 監視を有効化

1. Projectウィンドウでアクションアセットを選択
2. Inspectorで`Show In Monitor`を有効化
3. Monitor Windowを開く（Window > Reactive SO > Monitor）
4. Play Modeに入りアクションをトリガー

### Consoleロギング

1. Inspectorで`Show In Console`を有効化
2. `OnExecute`メソッド内で`LogAction(string)`を呼び出す
3. Play Mode中にConsoleにメッセージが表示される

### カスタムログメッセージ

```csharp
protected override void OnExecute()
{
    // ロジックをここに

#if UNITY_EDITOR
    LogAction($"カスタムメッセージ {details}");
#endif
}
```

---

## 一般的なパターン

### 複数アクションの連続実行

```csharp
public class SequenceAction : ActionSO
{
    [SerializeField] private ActionSO[] actions;

    protected override void OnExecute()
    {
        foreach (var action in actions)
        {
            action?.Execute();
        }
    }
}
```

### 条件付きアクション

```csharp
public class ConditionalAction : ActionSO
{
    [SerializeField] private BoolVariableSO condition;
    [SerializeField] private ActionSO trueAction;
    [SerializeField] private ActionSO falseAction;

    protected override void OnExecute()
    {
        if (condition != null && condition.Value)
        {
            trueAction?.Execute();
        }
        else
        {
            falseAction?.Execute();
        }
    }
}
```

### ランダムアクション

```csharp
public class RandomAction : ActionSO
{
    [SerializeField] private ActionSO[] actions;

    protected override void OnExecute()
    {
        if (actions.Length > 0)
        {
            int index = Random.Range(0, actions.Length);
            actions[index]?.Execute();
        }
    }
}
```

---

## ベストプラクティス

### エディタコードは常にラップ

```csharp
protected override void OnExecute()
{
    // ランタイムロジックをここに

#if UNITY_EDITOR
    // ロギングはエディタのみ（Monitor通知はExecute()が自動的に行う）
    LogAction("詳細");
#endif
}
```

### null条件演算子を使用

```csharp
// 安全な実行
action?.Execute();
```

### アクションを単一責任に保つ

1つのアクションは1つのことを行うべき。複雑な動作はシンプルなアクションの組み合わせで構成。

### descriptionフィールドで文書化

Inspectorで`description`フィールドを設定してアクションの動作を説明。

---

## 参照

- [Actionsガイド]({{ '/ja/guides/actions' | relative_url }}) - アクションの使い方
- [Monitor Window]({{ '/ja/debugging/monitor' | relative_url }}) - アクション実行のデバッグ
- [デバッグ概要]({{ '/ja/debugging/' | relative_url }}) - 全デバッグツール
