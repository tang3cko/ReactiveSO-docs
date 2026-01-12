---
layout: default
title: Dependency Analyzer
parent: デバッグ
nav_order: 3
---

# Dependency analyzer

## 目的

このガイドでは、静的解析のためのDependency Analyzerの使用方法を説明します。未使用のアセットの発見、未割り当てフィールドの検出、リファクタリングの影響の理解方法を学びます。

---

## ウィンドウを開く

**Window > Reactive SO > Dependency Analyzer**に移動します。

![Dependency Analyzer]({{ '/assets/images/debugging/dependency-analyzer.png' | relative_url }})

---

## タブ

Dependency Analyzerには4つのタブがあります。

| タブ | 説明 |
|-----|-------------|
| Event Channels | イベントチャンネルの使用状況と未割り当てフィールドを分析 |
| Actions | アクションの使用状況と未割り当てフィールドを分析 |
| Variables | 変数の使用状況と未割り当てフィールドを分析 |
| Runtime Sets | ランタイムセットの使用状況と未割り当てフィールドを分析 |

すべてのタブは、それぞれのアセットタイプに対して同じ機能を提供します。

---

## ユースケース

### 未割り当てフィールドの検出

**問題**
PlayerControllerを追加しましたが、イベントチャンネルの割り当てを忘れました。ゲームがランタイムでNullReferenceExceptionでクラッシュします。

**手順**

1. Dependency Analyzerを開く
2. **Scan Project**をクリック
3. Summary行で未割り当てフィールドを確認
4. Unassigned Event Channel Fieldsセクションを確認
5. 矢印をクリックして各場所にジャンプ

**例**

```text
Summary: 20 Event Channels, 1 unused, 3 unassigned fields

Unassigned Event Channel Fields:
  MainScene.unity > Player.PlayerController.onPlayerDeath
  EnemyPrefab.prefab > Enemy.EnemyAI.spawnEvent
  UICanvas.prefab > HealthBar.onHealthChanged
```

### 未使用のアセットを見つける

**問題**
プロジェクトに多くのアセットがあり、どれが未使用かわかりません。

**手順**

1. Dependency Analyzerを開く
2. **Scan Project**をクリック
3. **Show Unused Only**を有効化
4. リストを確認

使用箇所がゼロのアセットは安全に削除できます。

### リファクタリングの影響を確認

**問題**
`OnPlayerDeath`を削除する前に、どこで使用されているかを知る必要があります。

**手順**

1. Dependency Analyzerを開く
2. **Scan Project**をクリック
3. "OnPlayerDeath"を検索
4. アセットを展開してすべての使用箇所を表示
5. 矢印をクリックして各場所にジャンプ

**例**

```text
OnPlayerDeath (VoidEventChannelSO) - 5 usage(s)
  MainScene.unity
    Path: Assets/Scenes/MainScene.unity
    GameObject: Player
    Component: PlayerController
    Field: onPlayerDeath
  BossController.prefab
  GameOverUI.prefab
```

### 使用状況の文書化

**問題**
チームメンバーが「このイベントチャンネルはどこで使用されていますか?」と繰り返し尋ねます。

**手順**

1. Dependency Analyzerを開く
2. **Scan Project**をクリック
3. **Export** > Export to Markdownをクリック
4. レポートを保存
5. チームと共有するか、バージョン管理にコミット

---

## ウィンドウの機能

### Summary行

```text
Summary: 15 Event Channels, 2 unused, 3 unassigned fields
```

| 統計 | 説明 |
|------|-------------|
| アセットタイプ | プロジェクト内の総数 |
| unused | 参照がないアセット |
| unassigned fields | 宣言されているが割り当てられていないフィールド |

### ツリービュー

アセットはツリー構造で表示されます。

```text
OnPlayerDeath (VoidEventChannelSO) - 3 usage(s)
  MainScene.unity
    Path: Assets/Scenes/MainScene.unity
    GameObject: Player
    Component: PlayerController
    Field: onPlayerDeath
  BossController.prefab
  GameOverUI.prefab
```

矢印をクリックしてアセットにジャンプします。

### 未割り当てフィールドセクション

```text
Unassigned Event Channel Fields:

  MainScene.unity > Player.PlayerController.onPlayerDeath
  EnemyPrefab.prefab > Enemy.EnemyAI.spawnEvent
```

矢印をクリックしてジャンプして修正します。

---

## 検索とフィルタ

検索バーを使用して、名前、型、または部分一致で絞り込めます。

### Show Unused Only

参照がゼロのアセットのみを表示するには有効にします。削除可能なアセットを特定するために使用します。

---

## 結果のエクスポート

**Export**をクリックして形式を選択します。

| 形式 | ユースケース |
|--------|----------|
| Markdown | 人間が読みやすいドキュメント |
| JSON | プログラムによる分析 |
| CSV | スプレッドシート分析 |

### Markdownの例

```markdown
# Event Channels Dependency Report

Generated: 2025-12-27 10:30:00

## Summary

- Total Event Channels: 15
- Used Event Channels: 13
- Unused Event Channels: 2

### OnPlayerSpawn (VoidEventChannelSO)
**Path:** Assets/EventChannels/OnPlayerSpawn.asset
**Usages:** 3

1. **Assets/Scenes/MainScene.unity**
   - GameObject: SpawnManager
   - Component: SpawnManager
   - Field: spawnEventChannel
```

### JSONの例

```json
{
  "GeneratedAt": "2025-12-27T10:30:00Z",
  "Summary": {
    "TotalEventChannels": 15,
    "UsedEventChannels": 13,
    "UnusedEventChannels": 2
  },
  "EventChannels": [
    {
      "Name": "OnPlayerSpawn",
      "Type": "VoidEventChannelSO",
      "UsageCount": 3,
      "Usages": [...]
    }
  ]
}
```

### CSVの例

```csv
EventChannelName,EventChannelType,AssetPath,UsageCount,IsUsed
OnPlayerSpawn,VoidEventChannelSO,Assets/EventChannels/OnPlayerSpawn.asset,3,TRUE
OnPlayerDeath,VoidEventChannelSO,Assets/EventChannels/OnPlayerDeath.asset,0,FALSE
```

---

## 制限事項

Dependency Analyzerは静的解析のみを使用します。以下は検出しません。

- 動的参照 (`Resources.Load`、`Addressables.LoadAssetAsync`)
- 文字列ベースの参照
- ランタイムで作成された参照

ランタイム参照を検出するには、Play Mode中に[Monitor Window](monitor)を使用してください。

---

## パフォーマンス

スキャン時間はプロジェクトサイズに依存します:

| アセット数 | 時間 |
|--------|------|
| 100 | 約5秒 |
| 500 | 約15秒 |
| 1000 | 約30秒 |

プログレスバーにスキャン状態が表示されます。必要に応じてキャンセルできます。

---

## ヒント

- 大規模なリファクタリングの前に実行する
- Markdownにエクスポートしてバージョン管理にコミット
- 定期的にShow Unused Onlyを使用してプロジェクトをクリーンに保つ
- Event Monitorと組み合わせて静的および動的解析の両方を行う

---

## 参考資料

- [デバッグ概要]({{ '/ja/debugging/' | relative_url }}) - すべてのデバッグツール
- [Monitor Window](monitor) - ランタイム追跡
- [トラブルシューティング]({{ '/ja/troubleshooting' | relative_url }}) - 一般的な問題