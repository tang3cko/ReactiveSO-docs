---
layout: default
title: Event Types
parent: リファレンス
nav_order: 1
---

# イベントタイプ

## 目的

このリファレンスでは、12種類のビルトインイベントチャンネルタイプをすべて解説します。各タイプのユースケース、コード例、作成手順を確認できます。

---

## タイプ概要

| タイプ | パラメータ | ユースケース例 |
|------|-----------|-------------------|
| Void | なし | ゲーム開始、プレイヤー死亡、レベルクリア |
| Int | `int` | スコア変更、レベルアップ、敵の数 |
| Long | `long` | タイムスタンプ、大きな数値 |
| Float | `float` | ヘルスパーセント、タイマー、音量 |
| Double | `double` | 高精度値 |
| Bool | `bool` | ゲームポーズ、機能有効化 |
| String | `string` | 会話、通知、名前 |
| Vector2 | `Vector2` | 入力軸、タッチ位置 |
| Vector3 | `Vector3` | スポーン位置、ターゲット位置 |
| Quaternion | `Quaternion` | カメラ回転、方向 |
| Color | `Color` | テーマカラー、UIティント |
| GameObject | `GameObject` | 敵スポーン、ターゲット変更 |

---

## 適切なタイプの選択

適切なタイプを選択するために、以下の判断ガイドを使用してください。

1. **データが不要？** Voidを使用
2. **整数？**
   - 標準範囲（-20億〜+20億）: Intを使用
   - 大きな数値またはタイムスタンプ: Longを使用
3. **小数？**
   - 標準精度: Floatを使用
   - 高精度: Doubleを使用
4. **True/False？** Boolを使用
5. **テキスト？** Stringを使用
6. **2D位置/方向？** Vector2を使用
7. **3D位置/方向？** Vector3を使用
8. **3D回転？** Quaternionを使用
9. **色の値？** Colorを使用
10. **オブジェクト参照？** GameObjectを使用
11. **複雑なデータ？** カスタムタイプを作成

---

## Void

パラメータなしのイベント。イベント自体がすべての必要な情報を持っている場合に使用します。

### 作成

```text
Create > Reactive SO > Channels > Void Event
```

### 使用方法

```csharp
[SerializeField] private VoidEventChannelSO onPlayerDeath;

// Publisher
public void Die()
{
    onPlayerDeath?.RaiseEvent();
}

// Subscriber
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
    ShowGameOverScreen();
}
```

### 一般的なユースケース

- プレイヤーの死亡 / リスポーン
- ゲームのポーズ / 再開
- レベルのクリア / 失敗
- メニューの開閉
- アチーブメントのアンロック
- 保存完了

---

## Int

カウント、スコア、インデックスのための整数値を運ぶイベント。

### 作成

```text
Create > Reactive SO > Channels > Int Event
```

### 使用方法

```csharp
[SerializeField] private IntEventChannelSO onScoreChanged;

// Publisher
public void AddScore(int points)
{
    currentScore += points;
    onScoreChanged?.RaiseEvent(currentScore);
}

// Subscriber
private void OnEnable()
{
    onScoreChanged.OnEventRaised += UpdateScoreDisplay;
}

private void OnDisable()
{
    onScoreChanged.OnEventRaised -= UpdateScoreDisplay;
}

private void UpdateScoreDisplay(int newScore)
{
    scoreText.text = $"Score: {newScore}";
}
```

### 一般的なユースケース

- スコア更新
- 通貨変更（ゴールド、ジェム）
- インベントリカウント
- レベル / 経験値
- ウェーブ / ラウンド番号
- 敵の撃破数
- エンティティID

---

## Long

大きな数値とタイムスタンプのための64ビット整数値を運ぶイベント。

### 作成

```text
Create > Reactive SO > Channels > Long Event
```

### 使用方法

```csharp
[SerializeField] private LongEventChannelSO onTimestamp;

// Publisher
public void UpdateTimestamp()
{
    long timestamp = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();
    onTimestamp?.RaiseEvent(timestamp);
}

// Subscriber
private void OnEnable()
{
    onTimestamp.OnEventRaised += HandleTimestamp;
}

private void OnDisable()
{
    onTimestamp.OnEventRaised -= HandleTimestamp;
}

private void HandleTimestamp(long timestamp)
{
    Debug.Log($"Timestamp: {timestamp}");
}
```

### 一般的なユースケース

- Unixタイムスタンプ
- 大きなスコア値（20億超）
- ユニークID生成
- 長時間セッションでのフレームカウンタ
- ファイルサイズ
- データベースキー

---

## Float

パーセンテージ、ヘルス、時間のための浮動小数点値を運ぶイベント。

### 作成

```text
Create > Reactive SO > Channels > Float Event
```

### 使用方法

```csharp
[SerializeField] private FloatEventChannelSO onHealthChanged;

// Publisher
public void TakeDamage(float damage)
{
    currentHealth -= damage;
    float healthPercent = currentHealth / maxHealth;
    onHealthChanged?.RaiseEvent(healthPercent);
}

// Subscriber
private void OnEnable()
{
    onHealthChanged.OnEventRaised += UpdateHealthBar;
}

private void OnDisable()
{
    onHealthChanged.OnEventRaised -= UpdateHealthBar;
}

private void UpdateHealthBar(float healthPercent)
{
    healthBar.fillAmount = healthPercent;
}
```

### 一般的なユースケース

- ヘルス / スタミナ / マナ（0-1に正規化）
- プログレスバー（ローディング、クラフト）
- タイマーカウントダウン
- 音量 / ピッチ設定
- スピード倍率
- 温度値

---

## Double

高精度計算のための倍精度値を運ぶイベント。

### 作成

```text
Create > Reactive SO > Channels > Double Event
```

### 使用方法

```csharp
[SerializeField] private DoubleEventChannelSO onPreciseValue;

// Publisher
public void CalculateDistance()
{
    double distance = Vector3.Distance(pointA, pointB);
    onPreciseValue?.RaiseEvent(distance);
}

// Subscriber
private void OnEnable()
{
    onPreciseValue.OnEventRaised += HandleValue;
}

private void OnDisable()
{
    onPreciseValue.OnEventRaised -= HandleValue;
}

private void HandleValue(double value)
{
    Debug.Log($"Precise: {value:F10}");
}
```

### 一般的なユースケース

- 科学計算
- 金融値（高精度通貨）
- 大規模座標系
- 物理シミュレーション
- 統計計算

---

## Bool

状態トグルとフラグのためのブール値を運ぶイベント。

### 作成

```text
Create > Reactive SO > Channels > Bool Event
```

### 使用方法

```csharp
[SerializeField] private BoolEventChannelSO onGamePaused;

// Publisher
public void TogglePause()
{
    isPaused = !isPaused;
    onGamePaused?.RaiseEvent(isPaused);
}

// Subscriber
private void OnEnable()
{
    onGamePaused.OnEventRaised += HandlePause;
}

private void OnDisable()
{
    onGamePaused.OnEventRaised -= HandlePause;
}

private void HandlePause(bool isPaused)
{
    Time.timeScale = isPaused ? 0f : 1f;
}
```

### 一般的なユースケース

- ポーズ / 再開状態
- オーディオのミュート / ミュート解除
- ドアのロック / アンロック
- 機能の有効化 / 無効化
- 昼 / 夜サイクル
- パワーアップのアクティブ状態

---

## String

メッセージ、名前、識別子のためのテキストデータを運ぶイベント。

### 作成

```text
Create > Reactive SO > Channels > String Event
```

### 使用方法

```csharp
[SerializeField] private StringEventChannelSO onDialogue;

// Publisher
public void StartDialogue(string dialogueKey)
{
    onDialogue?.RaiseEvent(dialogueKey);
}

// Subscriber
private void OnEnable()
{
    onDialogue.OnEventRaised += ShowDialogue;
}

private void OnDisable()
{
    onDialogue.OnEventRaised -= ShowDialogue;
}

private void ShowDialogue(string dialogueKey)
{
    string text = dialogueDatabase.GetText(dialogueKey);
    dialogueBox.Display(text);
}
```

### 一般的なユースケース

- 会話 / カットシーントリガー
- クエスト名 / 説明
- アイテム名
- プレイヤー名
- ステータスメッセージ
- エラーメッセージ
- ローディング用のシーン名

---

## Vector2

位置、入力、スクリーン座標のための2Dベクトルデータを運ぶイベント。

### 作成

```text
Create > Reactive SO > Channels > Vector2 Event
```

### 使用方法

```csharp
[SerializeField] private Vector2EventChannelSO onInputAxis;

// Publisher
public void OnMove(InputAction.CallbackContext context)
{
    Vector2 input = context.ReadValue<Vector2>();
    onInputAxis?.RaiseEvent(input);
}

// Subscriber
private void OnEnable()
{
    onInputAxis.OnEventRaised += HandleMovement;
}

private void OnDisable()
{
    onInputAxis.OnEventRaised -= HandleMovement;
}

private void HandleMovement(Vector2 input)
{
    Vector3 movement = new Vector3(input.x, 0, input.y);
    transform.Translate(movement * speed * Time.deltaTime);
}
```

### 一般的なユースケース

- 2D移動入力（WASD、ジョイスティック）
- 2Dワールド位置
- UI要素の位置
- タッチ / マウスのスクリーン位置
- ミニマップ座標
- 2D速度

---

## Vector3

位置、方向、速度のための3Dベクトルデータを運ぶイベント。

### 作成

```text
Create > Reactive SO > Channels > Vector3 Event
```

### 使用方法

```csharp
[SerializeField] private Vector3EventChannelSO onSpawnPosition;

// Publisher
public void RequestSpawn()
{
    Vector3 position = GetRandomSpawnPoint();
    onSpawnPosition?.RaiseEvent(position);
}

// Subscriber
private void OnEnable()
{
    onSpawnPosition.OnEventRaised += SpawnEnemy;
}

private void OnDisable()
{
    onSpawnPosition.OnEventRaised -= SpawnEnemy;
}

private void SpawnEnemy(Vector3 position)
{
    Instantiate(enemyPrefab, position, Quaternion.identity);
}
```

### 一般的なユースケース

- スポーン位置
- ターゲット / ウェイポイント位置
- 視線方向
- 速度 / 力
- 衝撃点（弾丸、爆発）
- カメラ位置
- テレポート先

---

## Quaternion

3D方向のための回転値を運ぶイベント。

### 作成

```text
Create > Reactive SO > Channels > Quaternion Event
```

### 使用方法

```csharp
[SerializeField] private QuaternionEventChannelSO onCameraRotation;

// Publisher
public void RotateCamera()
{
    Quaternion rotation = cameraTransform.rotation;
    onCameraRotation?.RaiseEvent(rotation);
}

// Subscriber
private void OnEnable()
{
    onCameraRotation.OnEventRaised += HandleRotation;
}

private void OnDisable()
{
    onCameraRotation.OnEventRaised -= HandleRotation;
}

private void HandleRotation(Quaternion rotation)
{
    targetObject.rotation = rotation;
}
```

### 一般的なユースケース

- カメラ回転の同期
- オブジェクト方向の変更
- キャラクターの視線回転
- 手続き的回転
- ネットワーク回転の同期

---

## Color

UIテーマとビジュアルエフェクトのための色値を運ぶイベント。

### 作成

```text
Create > Reactive SO > Channels > Color Event
```

### 使用方法

```csharp
[SerializeField] private ColorEventChannelSO onThemeColor;

// Publisher
public void ChangeTheme(Color newColor)
{
    onThemeColor?.RaiseEvent(newColor);
}

// Subscriber
private void OnEnable()
{
    onThemeColor.OnEventRaised += HandleColorChange;
}

private void OnDisable()
{
    onThemeColor.OnEventRaised -= HandleColorChange;
}

private void HandleColorChange(Color color)
{
    uiPanel.color = color;
}
```

### 一般的なユースケース

- UIテーマ変更
- ヘルスバーの色インジケーター
- マテリアルティントの同期
- ビジュアルエフェクトの色
- チームカラー割り当て
- ライティングカラーの変更

---

## GameObject

スポーンされたエンティティとターゲットのためのオブジェクト参照を運ぶイベント。

### 作成

```text
Create > Reactive SO > Channels > GameObject Event
```

### 使用方法

```csharp
[SerializeField] private GameObjectEventChannelSO onEnemySpawned;

// Publisher
public void SpawnEnemy()
{
    GameObject enemy = Instantiate(enemyPrefab);
    onEnemySpawned?.RaiseEvent(enemy);
}

// Subscriber
private void OnEnable()
{
    onEnemySpawned.OnEventRaised += TrackEnemy;
}

private void OnDisable()
{
    onEnemySpawned.OnEventRaised -= TrackEnemy;
}

private void TrackEnemy(GameObject enemy)
{
    if (enemy != null)
    {
        activeEnemies.Add(enemy);
    }
}
```

### 一般的なユースケース

- スポーンされたオブジェクトの通知
- ターゲットの取得 / 喪失
- 選択されたオブジェクトの変更
- 拾ったアイテム
- トリガーされたオブジェクト
- プレイヤー / 敵の参照
- カメラターゲット

### 注意事項

- GameObjectチャンネルは「オブジェクトなし」を示すために`null`を渡すことができます
- 使用する前に、受信したGameObjectを常にnullチェックしてください
- オブジェクトは発行と受信の間に破棄される可能性があります

---

## カスタムタイプの作成

ビルトインタイプがニーズに合わない場合は、カスタムイベントタイプを作成できます。

```csharp
using Tang3cko.ReactiveSO;
using UnityEngine;

[CreateAssetMenu(
    fileName = "WeaponEvent",
    menuName = "Reactive SO/Channels/Weapon Event"
)]
public class WeaponEventChannelSO : EventChannelSO<WeaponData>
{
    // RaiseEvent(WeaponData value) が自動的に継承されます
}

[System.Serializable]
public struct WeaponData
{
    public string Name;
    public int Damage;
    public float Range;
}
```

カスタムタイプはEvent MonitorとSubscribers Listで動作しますが、Manual Triggerサポートはありません。

---

## 参考文献

- [Event Channelsガイド]({{ '/ja/guides/event-channels' | relative_url }}) - イベントチャンネルの使用方法
- [Variablesガイド]({{ '/ja/guides/variables' | relative_url }}) - リアクティブな状態管理
- [Debugging概要]({{ '/ja/debugging/' | relative_url }}) - イベントフローのデバッグ
