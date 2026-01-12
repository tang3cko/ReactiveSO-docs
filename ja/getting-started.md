---
layout: default
title: はじめに
nav_order: 2
---

# はじめに

---

## 目的

このガイドでは、Reactive SOをインストールし、最初のEvent Channelを作成する方法を説明します。ゲームシステムを疎結合化するための基本的なワークフローを学びます。

---

## インストール

Reactive SOはUnity Package Manager経由でインストールします。

1. **Window > Package Manager** を開く
2. パッケージリストで **Reactive SO** を探す
3. **Install** をクリック

[Asset Store](https://assetstore.unity.com/packages/tools/game-toolkits/reactive-so-339926)で購入した場合は、**My Assets** タブからパッケージをインポートしてください。

---

## クイックスタート - 最初のEvent Channel

### ステップ1: Event Channelアセットを作成

Projectウィンドウで右クリックし、以下のメニューパスを選択します。

```text
Create > Reactive SO > Channels > Void Event
```

![Create Menu]({{ '/assets/images/getting-started/create-menu.png' | relative_url }})

名前を `OnPlayerDeath` にします。

### ステップ2: イベントを発火（Publisher）

```csharp
using Tang3cko.ReactiveSO;
using UnityEngine;

public class Player : MonoBehaviour
{
    [SerializeField] private VoidEventChannelSO onPlayerDeath;

    public void Die()
    {
        onPlayerDeath?.RaiseEvent();
    }
}
```

### ステップ3: イベントを購読（Subscriber）

```csharp
using Tang3cko.ReactiveSO;
using UnityEngine;

public class GameManager : MonoBehaviour
{
    [SerializeField] private VoidEventChannelSO onPlayerDeath;

    private void OnEnable()
    {
        onPlayerDeath.OnEventRaised += HandlePlayerDeath;
    }

    private void OnDisable()
    {
        onPlayerDeath.OnEventRaised -= HandlePlayerDeath;
    }

    private void HandlePlayerDeath()
    {
        Debug.Log("Game Over!");
    }
}
```

### ステップ4: Inspectorで接続

1. Player GameObjectを選択
2. `OnPlayerDeath` アセットをシリアライズフィールドにドラッグ
3. GameManagerにも同様に設定

![Inspector Assignment]({{ '/assets/images/getting-started/inspector-assignment.png' | relative_url }})

これで完了です！PlayerとGameManagerは疎結合になりました。

---

## 利用可能なイベントタイプ

| タイプ | 用途 | 例 |
|--------|------|-----|
| Void | データなしの通知 | OnGameStart, OnPlayerDeath |
| Int | 整数値 | OnScoreChanged, OnLevelUp |
| Float | 浮動小数点値 | OnHealthChanged, OnProgress |
| Bool | 真偽値 | OnPaused, OnMuted |
| String | テキストメッセージ | OnDialogue, OnNotification |
| Vector2 | 2D位置/方向 | OnInputAxis, OnTouchPosition |
| Vector3 | 3D位置/方向 | OnSpawnPosition, OnTargetPosition |
| Quaternion | 回転 | OnCameraRotation |
| Color | 色 | OnThemeChanged |
| GameObject | オブジェクト参照 | OnEnemySpawned, OnTargetChanged |
| Long | 大きな整数 | OnTimestamp |
| Double | 高精度小数 | OnPreciseValue |

詳細は[Event Typesリファレンス]({{ '/ja/reference/event-types' | relative_url }})を参照してください。

---

## ベストプラクティス

### 必ず購読解除する

`OnDisable`で購読解除してメモリリークを防止しましょう。

```csharp
private void OnEnable()
{
    eventChannel.OnEventRaised += HandleEvent;
}

private void OnDisable()
{
    eventChannel.OnEventRaised -= HandleEvent;  // 必須！
}
```

### null条件演算子を使用

null参照例外を回避しましょう。

```csharp
// 良い例
onPlayerDeath?.RaiseEvent();

// 悪い例 - 未割り当て時に例外
onPlayerDeath.RaiseEvent();
```

### Inspectorで割り当て

イベントフローを可視化するために、コードで検索するのではなくInspectorでチャンネルを割り当てましょう。

---

## 動作を確認する

Monitor Windowを使って、イベントがリアルタイムで発火するのを確認しましょう。

1. **Window > Reactive SO > Monitor** を開く
2. Play Modeに入る
3. プレイヤーの死亡をトリガー——イベントがログに即座に表示される

この観察可能性がReactive SOの中核です。アーキテクチャで何が起きているかを常に確認できます。

[デバッグツールの詳細]({{ '/ja/debugging/' | relative_url }})

---

## 次のステップ

| やりたいこと | 参照 |
|--------------|------|
| 全イベントタイプを学ぶ | [Event Typesリファレンス]({{ '/ja/reference/event-types' | relative_url }}) |
| システム間で状態を共有 | [Variablesガイド]({{ '/ja/guides/variables' | relative_url }}) |
| オブジェクトコレクションを追跡 | [Runtime Setsガイド]({{ '/ja/guides/runtime-sets' | relative_url }}) |
| エンティティ状態を管理 | [Reactive Entity Setsガイド]({{ '/ja/guides/reactive-entity-sets' | relative_url }}) |
| イベントフローをデバッグ | [デバッグ]({{ '/ja/debugging/' | relative_url }}) |
| アーキテクチャを理解 | [Event Channelsガイド]({{ '/ja/guides/event-channels' | relative_url }}) |
