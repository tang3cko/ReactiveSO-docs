---
layout: default
title: ホーム
nav_order: 1
---

# Reactive SO

Unity向けのScriptableObjectベースのリアクティブアーキテクチャ。Event Channels、Variables、Runtime Sets、Reactive Entity Setsを提供します。

<iframe width="560" height="315" src="https://www.youtube.com/embed/8it_Js-iabI" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

---

## 目的

このドキュメントは、UnityプロジェクトでReactive SOを使用するために必要なすべてを網羅しています。インストール、設定、各機能の効果的な使い方を学びます。

---

## なぜReactive SO？

従来のUnity開発では、コンポーネント間が密結合になりがちで、テストや保守が困難になります。Reactive SOはこれらの課題を4つのコア原則で解決します。

### 設計レベルでの疎結合

パブリッシャーとサブスクライバーはScriptableObjectアセットを介して通信します。シングルトンもFindObjectOfTypeも密結合もありません。

### Inspector中心

イベントの接続、共有状態の設定、依存関係の確認——すべてInspectorで完結。コードを触らずにアーキテクチャを可視化・編集できます。

### 観察可能

実行中の状態をリアルタイムで確認できます。Monitor Windowはイベントの発火、変数の変更、セットの更新を表示。Asset BrowserとDependency Analyzerでプロジェクトの理解と保守をサポートします。

### スケーラブル

Event Channelsでシンプルな通知から始めましょう。Variablesで状態共有、Runtime Setsでオブジェクト追跡。Reactive Entity Setsなら数万のエンティティをJob System統合で管理できます。

---

## 主な機能

### Event Channels

ScriptableObjectアセットを使用したグローバルイベント通知。

```csharp
// イベントを発火
onPlayerDeath?.RaiseEvent();

// イベントを購読
onPlayerDeath.OnEventRaised += HandlePlayerDeath;
```

[Event Channelsの詳細]({{ '/ja/guides/event-channels' | relative_url }})

### Variables

自動変更検出付きのリアクティブな共有状態。

```csharp
// 値の変更で自動的にイベントが発火
playerHealth.Value = 100;
playerHealth.OnValueChanged += UpdateHealthUI;
```

[Variablesの詳細]({{ '/ja/guides/variables' | relative_url }})

### Runtime Sets

シングルトンなしで動的オブジェクトコレクションを管理。

```csharp
// シーン内の全敵を追跡
enemySet.Add(this.gameObject);
foreach (var enemy in enemySet.Items) { ... }
```

[Runtime Setsの詳細]({{ '/ja/guides/runtime-sets' | relative_url }})

### Reactive Entity Sets

シーンに依存しない集中型エンティティ状態管理。

```csharp
// シーンに依存しないO(1)アクセスの状態管理
entitySet.Register(entityId, new EntityData { ... });
```

[Reactive Entity Setsの詳細]({{ '/ja/guides/reactive-entity-sets' | relative_url }})

---

## 追加機能

### ActionSO

ScriptableObjectアセットとして実装されたCommandパターン。再利用可能なアクションを定義し、Inspectorで設定、Play Mode中に手動実行してテストできます。

[Actionsの詳細]({{ '/ja/guides/actions' | relative_url }})

### R3連携

[R3](https://github.com/Cysharp/R3)がインストールされている場合、Event ChannelsとReactive Entity SetsをObservableストリームに変換し、リアクティブプログラミングのワークフローに統合できます。

[Integrationsの詳細]({{ '/ja/integrations/' | relative_url }})

---

## はじめに

Reactive SOをプロジェクトで使用する準備はできましたか？

[はじめる]({{ '/ja/getting-started' | relative_url }}){: .btn .btn-primary }

---

## 動作要件

- **v2.1.0以降**: Unity 6000.0 (Unity 6.0) 以上
- v1.1.1–v2.0.0: Unity 6000.2 (Unity 6.2) 以上
- v1.0.0–v1.1.0: Unity 2022.3 以上

---

## リンク

- [Asset Store](https://assetstore.unity.com/packages/tools/game-toolkits/reactive-so-339926)
- [ブログシリーズ](https://tang3cko.hashnode.dev/series/reactive-so)
- [English Documentation]({{ '/en/' | relative_url }})
