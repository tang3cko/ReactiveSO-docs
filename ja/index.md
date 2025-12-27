---
layout: default
title: ホーム
nav_order: 1
---

# Reactive SO

Unity向けのScriptableObjectベースのリアクティブアーキテクチャ。Event Channels、Variables、Runtime Sets、Reactive Entity Setsを提供します。

---

## 目的

このドキュメントは、UnityプロジェクトでReactive SOを使用するために必要なすべてを網羅しています。インストール、設定、各機能の効果的な使い方を学びます。

---

## なぜReactive SO？

従来のUnity開発では、コンポーネント間が密結合になりがちで、テストや保守が困難になります。Reactive SOはこれらの課題をコア設計原則によって解決します。

- **完全な疎結合** - パブリッシャーとサブスクライバーがScriptableObjectアセットを通じて直接参照なしに通信
- **視覚的なイベントフロー** - Unity Inspectorでイベントの依存関係を直接確認
- **ゼロランタイムオーバーヘッド** - デバッグ機能はEditorでのみ動作
- **強い型付け** - 一般的なUnity型をカバーする12種類のイベントタイプ

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

## はじめに

Reactive SOをプロジェクトで使用する準備はできましたか？

[はじめる]({{ '/ja/getting-started' | relative_url }}){: .btn .btn-primary }

---

## 動作要件

- Unity 6000.2以降

---

## リンク

- [Asset Store](https://assetstore.unity.com/packages/tools/game-toolkits/reactive-so-339926)
- [English Documentation]({{ '/en/' | relative_url }})
