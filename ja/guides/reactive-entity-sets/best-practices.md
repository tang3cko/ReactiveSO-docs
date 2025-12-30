---
layout: default
title: ベストプラクティス
parent: Reactive Entity Sets
grand_parent: ガイド
nav_order: 4
---

# ベストプラクティス

---

## 目的

このページでは、Reactive Entity Setsを効果的に使用するためのベストプラクティスを説明します。state構造体の設計、更新パターン、メモリ管理、一般的な問題のトラブルシューティングについて学びます。

---

## state構造体の設計

### state構造体をシンプルに保つ

最高のパフォーマンスのためにプリミティブ型を使用します。

```csharp
// 良い: シンプルな値型
[Serializable]
public struct EntityState
{
    public int Health;
    public float Speed;
    public Vector3 LastPosition;
}
```

```csharp
// 悪い: 参照型はアロケーションを引き起こす
[Serializable]
public struct EntityState
{
    public List<int> Modifiers;  // 避ける
    public string Name;           // 可能なら避ける
}
```

### 計算プロパティは問題なし

格納された値から派生する読み取り専用プロパティは許容されます。

```csharp
[Serializable]
public struct EntityState
{
    public int Health;
    public int MaxHealth;

    // 良い: 格納された値から計算
    public float HealthPercent => MaxHealth > 0 ? (float)Health / MaxHealth : 0f;
    public bool IsDead => Health <= 0;
}
```

---

## 更新パターン

### アトミックな変更にはUpdateDataを使用

```csharp
// 良い: アトミックな更新
entitySet.UpdateData(this, state => {
    state.Health -= damage;
    state.IsStunned = damage > 50;
    return state;
});
```

```csharp
// 悪い: 非アトミック、getとsetの間に状態が変わる可能性
var state = entitySet.GetData(this);
state.Health -= damage;
entitySet.SetData(this, state);
```

### 状態を直接変更しない

直接の構造体変更はセットを更新しません。

```csharp
// 悪い: 直接変更はイベントをトリガーしない
var state = entitySet.GetData(entityId);
state.Health -= 10;  // セットは更新されない！
```

```csharp
// 良い: SetDataまたはUpdateDataを使用
entitySet.UpdateData(entityId, state => {
    state.Health -= 10;
    return state;
});
```

---

## サブスクリプション管理

### 常にサブスクライブ解除

```csharp
// 良い: バランスの取れたサブスクリプション
private void OnEnable()
{
    entitySet.SubscribeToEntity(entityId, OnStateChanged);
}

private void OnDisable()
{
    entitySet.UnsubscribeFromEntity(entityId, OnStateChanged);
}
```

```csharp
// 悪い: メモリリーク
private void Start()
{
    entitySet.SubscribeToEntity(entityId, OnStateChanged);
}
// OnDisableでのサブスクライブ解除が欠落
```

### サブスクライブ時に状態を初期化

サブスクライブ直後にUIを更新します。

```csharp
private void OnEnable()
{
    entitySet.SubscribeToEntity(entityId, OnStateChanged);

    // 即座に初期化
    if (entitySet.TryGetData(entityId, out var state))
    {
        UpdateDisplay(state);
    }
}
```

---

## シーン永続化

### 状態はシーン変更を生き残る

エンティティデータはScriptableObjectに格納され、シーンロード間で永続化されます。

{: .warning }
> **重要**: ScriptableObjectはシーン遷移時にアクティブなオブジェクトから参照されていない場合、メモリからアンロードされる可能性があります。データ損失を防ぐには、Manager Sceneで`ReactiveEntitySetHolder`を使用してください。詳細は[永続化](persistence)を参照してください。

```csharp
// シーンA: 敵を登録
entitySet.Register(enemyId, initialState);

// シーンBがロード
// 状態はまだアクセス可能
if (entitySet.TryGetData(enemyId, out var state))
{
    // 状態が永続化
}
```

### 適切なタイミングでクリア

ゲームコンテキストが変わるときにクリーンアップします。

```csharp
public class GameManager : MonoBehaviour
{
    [SerializeField] private EnemyEntitySetSO entitySet;

    public void RestartLevel()
    {
        entitySet.Clear();
        SceneManager.LoadScene(SceneManager.GetActiveScene().name);
    }
}
```

### Play Modeとビルドの動作

ScriptableObjectデータはプレイセッション中は永続化しますが、Play Modeを終了（エディタ）またはアプリケーションを再起動（ビルド）するとリセットされます。永続的な保存には、PlayerPrefs、JSONファイル、またはデータベースにシリアライズしてください。

---

## パフォーマンスガイドライン

### RESが優れている場合

- フレームあたりのエンティティ変更が10%未満
- エンティティ数が中程度（数百から数千程度）
- 複数のシステムが同じ状態変更に反応する必要がある

### 代替案を検討する場合

- すべてのエンティティが毎フレーム更新される
- エンティティ数が数万に達する
- 生のパフォーマンスが最優先事項

### 高頻度データを避ける

```csharp
// これが毎フレーム変更される場合は再検討
[Serializable]
public struct EntityState
{
    public float AnimationTime;  // 毎フレーム変更 - 避ける
    public Vector3 Velocity;     // 毎フレーム変更 - 避ける
}
```

詳細なガイダンスについては[データガイドライン]({{ '/ja/design-philosophy/reactive-entity-sets/data-guidelines' | relative_url }})を参照してください。

---

## トラブルシューティング

### 状態が更新されない

`SetData`または`UpdateData`を使用していることを確認してください。直接の構造体変更はセットを更新しません。

```csharp
// 間違い
var state = entitySet.GetData(entityId);
state.Health = 50;  // セットは更新されない！

// 正しい
entitySet.SetData(entityId, new EnemyState { Health = 50 });
```

### イベントが発火しない

1. `Contains()`でエンティティが登録されているか確認
2. 状態変更の前にサブスクライブしているか確認
3. Inspectorでイベントチャンネルが割り当てられているか確認

### エンティティが見つからない

1. エンティティをクエリする前に`OnEnable`が実行されているか確認
2. エンティティIDが正しいか確認（`GetInstanceID()`はプレイセッションごとに変わる）
3. 安全なアクセスには`GetData`の代わりに`TryGetData`を使用

```csharp
// 安全なアクセスパターン
if (entitySet.TryGetData(entityId, out var state))
{
    // stateを使用
}
else
{
    // エンティティが見つからない場合の処理
}
```

### メモリリーク

`OnDisable`で常にサブスクライブ解除してください。

```csharp
private void OnDisable()
{
    entitySet.UnsubscribeFromEntity(entityId, OnStateChanged);
}
```

### シーンロード後の古いサブスクリプション

シーンがアンロードされると、GameObjectは破棄されますがScriptableObjectデータは永続化されます。OnDisableでサブスクライブ解除しないと、コールバックが破棄されたオブジェクトを指す可能性があります。

```csharp
// 常にOnDisableでサブスクライブ解除
private void OnDisable()
{
    if (trackedEntityId != 0)
    {
        entitySet.UnsubscribeFromEntity(trackedEntityId, OnStateChanged);
    }
}
```

---

## まとめ

| プラクティス | 説明 |
|-------------|------|
| シンプルな構造体 | プリミティブ型を使用、参照型を避ける |
| アトミックな更新 | 複数フィールドの変更にはUpdateDataを使用 |
| バランスの取れたサブスクリプション | 常にOnDisableでサブスクライブ解除 |
| サブスクライブ時に初期化 | サブスクライブ直後に表示を更新 |
| 安全なアクセス | null許容アクセスにはTryGetDataを使用 |
| シーン認識 | ゲームコンテキストが変わるときにセットをクリア |

---

## 関連ドキュメント

- [RES設計思想]({{ '/ja/design-philosophy/reactive-entity-sets/' | relative_url }}) - 概念的な基礎
- [データガイドライン]({{ '/ja/design-philosophy/reactive-entity-sets/data-guidelines' | relative_url }}) - RESに含めるべきデータ
