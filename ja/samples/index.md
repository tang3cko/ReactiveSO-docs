---
layout: default
title: サンプル
nav_order: 5
has_children: true
---

# サンプル

このセクションでは、Reactive SOに含まれるサンプルプロジェクトについて解説します。これらのサンプルは、アーキテクチャパターンの実践的な適用例を示しています。

サンプルのソースコードはパッケージ内の `Samples~` フォルダに含まれています。プロジェクトにインストールするには、Unityの **Package Manager** ウィンドウを使用してください。

1. **Window > Package Manager** を開く
2. リストから **Reactive SO** を選択
3. **Samples** セクションを展開
4. 試したいサンプルの横にある **Import** をクリック

---

## 利用可能なサンプル

| サンプル | 主な概念 | 説明 |
| :--- | :--- | :--- |
| [基本デモ (Counter)](basic-demo) | Event Channels, Variables | ボタン、ロジック、表示間の疎結合な通信を示すシンプルなカウンターUI。 |
| [GPU Sync (Motes)](mote-demo) | GPU Sync, Variables, Compute Shaders | VariableSOの値をシェーダーに直接同期させ、高性能な視覚効果を実現する方法を示します。 |
| [Runtime Sets デモ](runtime-sets-demo) | Runtime Sets, Dynamic Lists | マネージャークラス（シングルトン）を使わずに、動的なオブジェクトの集合（スポーンされた敵など）を管理する方法を示します。 |
