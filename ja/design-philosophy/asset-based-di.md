---
layout: default
title: Asset-based DI
parent: 設計思想
nav_order: 2.5
---

# アセットベースの依存性注入 (Asset-based DI)

---

## 目的

このページでは、**アセットベースの依存性注入（Asset-based Dependency Injection）** という概念について解説します。Reactive SOがなぜScriptableObjectを単なるデータコンテナとしてではなく、システム間の依存関係を解決する主要なメカニズムとして扱っているのかを学びます。

---

## 依存性注入（DI）とは何か？

アセットベースDIを理解する前に、依存性注入（Dependency Injection）が実際に何を意味するのかを明確にしましょう。

### 定義

[Martin Fowlerは2004年に依存性注入を定義しました](https://martinfowler.com/articles/injection.html)。

> 「依存性注入の基本的な考え方は、別個のオブジェクト（アセンブラ）がリスタークラスのフィールドに適切な実装を設定することである。」

簡単に言えば：**コンポーネントが依存関係を自分で作成するのではなく、外部から受け取る**パターンです。

### DIの3つの形式

| 形式 | 説明 | 例 |
|:-----|:-----|:---|
| **Constructor Injection** | コンストラクタ経由で依存を渡す | `new Service(dependency)` |
| **Setter Injection** | Setterメソッド経由で依存を設定 | `service.SetDependency(dep)` |
| **Field Injection** | フィールドに直接依存を設定 | `service.dependency = dep` |

### DI ≠ DIコンテナ

{: .important }
> **依存性注入**は設計パターンです。**DIコンテナ**（VContainerやZenjectなど）はDIを自動化するツールです。依存性注入を実践するためにDIコンテナは必要ありません。

この区別は非常に重要です。

| 用語 | 意味 |
|:-----|:-----|
| **依存性注入（DI）** | 依存関係を外部から提供する*パターン* |
| **DIコンテナ** | 依存解決とライフサイクル管理を自動化する*ツール* |
| **Pure DI / Manual DI** | コンテナを使わず、手動で依存関係を組み立てるDIの実践 |

多くの開発者が「DI = Zenject」や「DI = VContainer」と思い込んでいます。これはツールとパターンを混同しています。VContainerは内部でDIを*実行*しますが、VContainer自体はDIではありません。

### UnityにおけるPure DI

以下のコードを書いたとします。

```csharp
public class PlayerController : MonoBehaviour
{
    [SerializeField] private IntVariableSO health;  // 依存関係
}
```

そしてInspectorのフィールドにアセットをドラッグすると、あなたは**Pure DI（Field Injection経由）**を実践しています。Unity Inspectorが依存関係を設定する「アセンブラ」として機能します。DIコンテナは不要です。

---

## データコンテナを超えて

「ScriptableObjectは武器のステータスやゲーム設定などの静的なデータのためのもの」という誤解がよくあります。もちろんそれにも適していますが、真の力はその**ライフサイクル**と**アイデンティティ（同一性）**にあります。

ScriptableObjectは以下の特性を持ちます。
1.  **共有インスタンス** - シーンから独立してメモリ上に存在します。
2.  **アセット参照** - 固有のGUIDを持ち、Inspectorで割り当てることができます。

Reactive SOはこれらの特性を活用し、ScriptableObjectをシステム間の**動的な接続ポイント**として利用します。

---

## Unityにおける型の比較

各アプローチにはそれぞれの強みがあります。これらのトレードオフを理解することで、状況に応じて適切なツールを選択できます。

| 機能 | ScriptableObject | 静的クラス (Singleton) | MonoBehaviour | 純粋なC#クラス |
|:---|:---|:---|:---|:---|
| **ライフサイクル** | プロジェクトスコープ<br>(参照がある限り生存) | App Domain<br>(ドメインリロードでリセット) | シーンスコープ<br>(シーンライフサイクルに紐づく) | GCによって管理 |
| **依存性の注入** | Inspector (アセット)<br>ドラッグ＆ドロップ | 直接アクセス<br>`Manager.Instance` | Inspector (シーン)<br>または `GetComponent` | コンストラクタ / Factory |
| **テスト容易性** | テストで`CreateInstance`可能 | テスト間でリセットが必要 | GameObjectのセットアップが必要 | インスタンス化が容易 |
| **設定・調整** | シリアライズ<br>Inspectorで編集可能 | コードまたは設定ファイル | シリアライズ<br>シーン内のインスタンスごと | コードのみ |
| **多重性** | 複数のアセットが可能 | グローバルに単一インスタンス | 複数のGameObject | 複数のインスタンス |

### 使い分けの指針

| アプローチ | 適したユースケース |
|:---------|:----------------|
| **ScriptableObject** | シーン間で共有する状態、データ駆動の設定、疎結合な通信 |
| **静的クラス / Singleton** | 真にグローバルなサービス（ログ出力など）、シンプルなプロトタイプ、素早いアクセスパターン |
| **MonoBehaviour** | シーン固有の振る舞い、ライフサイクル管理が必要なGameObject、ビジュアルコンポーネント |
| **純粋なC#クラス** | ビジネスロジック、アルゴリズム、ドメインモデル、最大限のテスト容易性 |

### ScriptableObjectのトレードオフ

ScriptableObjectは万能ではありません。以下のトレードオフを考慮してください。

- **ランタイムデータの永続性** - 非シリアライズデータはアセットがアンロードされると失われる可能性があります（[トラブルシューティング]({{ '/ja/troubleshooting#data-loss' | relative_url }})を参照）
- **シーン固有ではない設計** - シーンごとの状態には、MonoBehaviourの方が適切な場合があります
- **アセット管理地獄（Asset Management Hell）** - これは現実です。このパターンを使った成熟したプロジェクトでは、数十から数百のアセットが生まれます：`OnPlayerDeath`、`OnEnemySpawned`、`PlayerHealth`、`EnemySet`... 適切なアセットを見つけ、重複を避け、命名規則を維持することは、本当の課題になります。Reactive SOは[Asset Browser]({{ '/ja/debugging/asset-browser' | relative_url }})や[Dependency Analyzer]({{ '/ja/debugging/dependency-analyzer' | relative_url }})などのツールを提供していますが、このオーバーヘッドは避けられません。

Reactive SOがScriptableObjectを選択するのは、**シーン間で共有される状態**と**疎結合な通信**に優れているからです - まさにこのアーキテクチャが解決しようとしている課題です。ただし、**コードの複雑さをアセットの複雑さと交換している**ことを認識してください。

## 概念：アセットベースの依存性注入

**アセットベースの依存性注入（Asset-based DI）**は**Inspector経由のPure DI**であり、クラスやシーン上のオブジェクトではなく、**アセット**への参照を通じて依存関係を解決するパターンです。

### 従来の方法（密結合）

```csharp
public class EnemySpawner : MonoBehaviour
{
    // 特定のシーンオブジェクトやシングルトンへの依存
    void Spawn()
    {
        // 「このシーンにある特定のManagerを探さなきゃいけない」
        GameManager.Instance.RegisterEnemy(newEnemy);
    }
}
```

このアプローチは`GameManager`への強い依存を作ります。`GameManager`なしでは`EnemySpawner`をテストできず、複数の敵リストを持つこともできません。

### アセットベースDI（疎結合）

```csharp
public class EnemySpawner : MonoBehaviour
{
    // Inspector経由で注入される依存（Field Injection）
    [SerializeField] private GameObjectRuntimeSetSO enemySet;

    void Spawn()
    {
        // 「誰がこのリストを持っているかは知らないけど、ここに追加すればいい」
        enemySet.Add(newEnemy);
    }
}
```

これは**Pure DI**です：依存関係（`enemySet`）はInspector経由で外部から提供されます。Unity InspectorはFowlerの用語でいう「アセンブラ」として機能し、DIコンテナフレームワークを必要とせずに依存関係を手動で配線します。

---

## ケーススタディ：DIとしてのRuntime Sets

[Runtime Sets]({{ '/ja/guides/runtime-sets' | relative_url }}) は、アセットベースDIの最も原始的で分かりやすい例です。

1.  **課題**: コードが「全ての敵のリスト」を必要としている。
2.  **素朴な解決策**: Managerクラスに `public static List<Enemy> AllEnemies` を作る。
    *   *問題点*: テストしにくい、複数のコンテキスト（ステージなど）の管理が難しい、隠れた依存関係が生まれる。
3.  **アセットベースの解決策**: `EnemySet` アセット（ScriptableObject）を作る。
    *   敵は自分自身をアセットに登録する：`set.Add(this)`
    *   システムはアセットから読み取る：`foreach (var e in set.Items)`

`EnemySet` アセットは、プロジェクトにおける**安定したアンカー（錨）**となります。シーンがロード/アンロードされようと、MonoBehaviourが生成/破棄されようと、**接続ポイント（アセット）**は不変です。

---

## なぜReactive SOはこれに着目するのか

Reactive SOは、アセットベースDIのメリットを最大化するように設計されています。

1.  **Event Channels** - メソッド呼び出し（`Action<T>`）として振る舞うアセット。
2.  **Variables** - 共有フィールド（`T value`）として振る舞うアセット。
3.  **Reactive Entity Sets** - インメモリデータベースとして振る舞うアセット。

これらの責務をアセットに移譲することで、MonoBehaviourは**ステートレスなロジックの入れ物**となり、記述、解読、そしてテストが容易になります。
