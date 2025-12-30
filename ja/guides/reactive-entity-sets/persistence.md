---
layout: default
title: 永続化
parent: Reactive Entity Sets
grand_parent: ガイド
nav_order: 5
---

# 永続化

---

## 目的

このページでは、シーン遷移時にReactiveEntitySetSOのデータが失われる理由と、Manager SceneパターンとReactiveEntitySetHolderを使用してそれを防ぐ方法を説明します。

---

## 問題

Unityは、どのシーンからも参照されていないScriptableObjectアセットをメモリからアンロードすることがあります。ReactiveEntitySetSOはランタイムデータを非シリアライズフィールドに保存するため、アセットがアンロードされて再ロードされるとこのデータは失われます。

### 症状

- シーン遷移前に登録したエンティティが、戻ってきた後に消えている
- エンティティを登録したにもかかわらず`Count`が0を返す
- Inspectorでアセットを選択している時だけデータが保持されているように見える（エディタのみ）

### なぜこれが起きるのか

ScriptableObjectはデータコンテナであり、メモリマネージャーではありません。シーン内のオブジェクトがSOを参照していない場合、Unityはメモリを解放するためにアンロードすることがあります。再ロード時、非シリアライズフィールドはデフォルト値にリセットされます。

---

## 解決策: Manager Sceneパターン

推奨される解決策は、シーン遷移後も永続するManager Sceneを使用し、ReactiveEntitySetSOアセットへの参照を保持することです。

### Manager Sceneとは

Manager Sceneは、追加ロードされ、アンロードされないシーンです。通常以下を含みます。

- ゲームマネージャー（GameManager、AudioManagerなど）
- 永続的なUI要素
- 永続化が必要なScriptableObjectアセットへの参照

### ステップ1: Manager Sceneを作成する

1. `ManagerScene`（または類似の名前）という新しいシーンを作成
2. `Managers`という名前のGameObjectを追加
3. このGameObjectにマネージャーコンポーネントを追加

### ステップ2: ReactiveEntitySetHolderを追加する

1. `RES References`という名前の空のGameObjectを作成
2. `ReactiveEntitySetHolder`コンポーネントを追加
3. ReactiveEntitySetSOアセットをreferencesリストに追加

```text
ManagerScene
└── Managers
└── RES References
    └── ReactiveEntitySetHolder
        └── references: [EnemyEntitySet, PlayerEntitySet, ...]
```

### ステップ3: 起動時にManager Sceneをロードする

ゲームのエントリーポイントで、Manager Sceneを追加ロードします:

```csharp
using UnityEngine;
using UnityEngine.SceneManagement;

public class GameBootstrap : MonoBehaviour
{
    [SerializeField] private string managerSceneName = "ManagerScene";

    private void Awake()
    {
        // まだロードされていなければManager Sceneをロード
        if (!SceneManager.GetSceneByName(managerSceneName).isLoaded)
        {
            SceneManager.LoadScene(managerSceneName, LoadSceneMode.Additive);
        }
    }
}
```

### ステップ4: ゲームシーンを追加ロードする

Manager Sceneをアンロードせずにゲームシーンをロードします:

```csharp
// 新しいレベルをロード
SceneManager.LoadScene("Level_01", LoadSceneMode.Additive);

// 前のレベルをアンロード（Manager Sceneはロードされたまま）
SceneManager.UnloadSceneAsync("Level_00");
```

---

## エディタユーティリティ

ReactiveEntitySetHolderには参照管理を支援するエディタボタンが含まれています。

| ボタン | 説明 |
|--------|------|
| **Find All in Project** | プロジェクト内のすべてのReactiveEntitySetSOアセットを検索してリストに追加 |
| **Clear Missing** | リストからnullエントリを削除（例: アセット削除後） |

---

## ベストプラクティス

1. **Manager Sceneは1つに** - 明確さのため、すべての永続的な参照を1つのシーンに保持
2. **参照は明示的に追加** - 「Find All in Project」は便利ですが、明示的にアセットを追加すると依存関係がより明確に
3. **DontDestroyOnLoadは使わない** - Manager Sceneパターンは、個々のオブジェクトをDontDestroyOnLoadとしてマークするよりクリーン

---

## 次のステップ

- [イベント](events) - エンティティおよびセットレベルの変更をサブスクライブ
- [パターン](patterns) - 実際のユースケースパターン
