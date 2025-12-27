---
layout: default
title: Variable Types
parent: リファレンス
nav_order: 2
---

# 変数タイプ

{: .note }
> 変数はv1.1.0以降で利用可能。GPU Syncはv2.0.0で追加されました。

## 目的

このリファレンスでは、11種類のビルトイン変数タイプをすべて解説します。各タイプのユースケース、GPU Sync互換性、コード例を確認できます。

---

## タイプ概要

| タイプ | GPU Sync | ユースケース例 |
|------|----------|-------------------|
| Int | ✅ | スコア、ヘルス、通貨、レベル |
| Long | ❌ | タイムスタンプ、大きな数値 |
| Float | ✅ | タイマー、音量、進行状況 |
| Double | ❌ | 高精度値 |
| Bool | ✅ | ゲームポーズ、機能有効化 |
| String | ❌ | プレイヤー名、ステータスメッセージ |
| Vector2 | ✅ | 2D位置、入力軸 |
| Vector3 | ✅ | 3D位置、速度 |
| Quaternion | ✅ | 回転、方向 |
| Color | ✅ | テーマカラー、マテリアルティント |
| GameObject | ❌ | 現在のターゲット、選択オブジェクト |

---

## GPU Syncサポート

GPU Syncにより、変数値がシェーダーグローバルプロパティに自動的に同期されます。これにより、シェーダー、VFX Graph、Compute Shaderが追加のブリッジコードなしでゲームプレイ状態に反応できます。

| 変数タイプ | シェーダーメソッド | HLSLタイプ |
|---------------|---------------|-----------|
| Int | `SetGlobalInteger` | `int` |
| Float | `SetGlobalFloat` | `float` |
| Bool | `SetGlobalInteger` | `int` (0または1) |
| Vector2 | `SetGlobalVector` | `float4` (xy使用) |
| Vector3 | `SetGlobalVector` | `float4` (xyz使用) |
| Quaternion | `SetGlobalVector` | `float4` (xyzw) |
| Color | `SetGlobalColor` | `float4` |

String、Long、Double、GameObjectはデータ型の都合上GPU Syncをサポートしていません。

---

## 共通API

すべての変数タイプは以下のAPIを共有します。

### プロパティ

| プロパティ | タイプ | 説明 |
|----------|------|-------------|
| Value | `T` | 現在の値 (get/set) |
| InitialValue | `T` | Inspectorで設定された値 |
| OnValueChanged | `EventChannelSO<T>` | 変更を通知するイベントチャンネル |

### メソッド

| メソッド | 説明 |
|--------|-------------|
| `ResetToInitial()` | ValueをInitialValueに復元 |
| `NotifyValueChanged()` | 値の変更なしでイベント通知を強制 |

### イベント

`Value`を設定すると、新しい値が現在の値と異なる場合に自動的に`OnValueChanged`が発火します。比較には`EqualityComparer<T>`が使用されます。

---

## Int

標準範囲の整数のための変数。

### 作成

```text
Create > Reactive SO > Variables > Int Variable
```

### 使用方法

```csharp
[SerializeField] private IntVariableSO playerScore;

// 書き込み
playerScore.Value += 100;

// 読み取り
int score = playerScore.Value;

// 変更をリッスン（リンクされたイベントチャンネル経由）
onScoreChanged.OnEventRaised += UpdateScoreUI;
```

### GPU Sync

```hlsl
// シェーダー内（GPU Sync有効化後）
int score = _PlayerScore;
```

### 一般的なユースケース

- プレイヤースコア
- 通貨（ゴールド、ジェム）
- ヘルス / スタミナ（整数ベース）
- レベル / 経験値
- ウェーブ / ラウンド番号
- インベントリ数量

---

## Long

64ビット整数変数。大きな数値とタイムスタンプ用。

### 作成

```text
Create > Reactive SO > Variables > Long Variable
```

### 使用方法

```csharp
[SerializeField] private LongVariableSO lastSaveTime;

// 書き込み
lastSaveTime.Value = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();

// 読み取り
long timestamp = lastSaveTime.Value;
```

### 一般的なユースケース

- Unixタイムスタンプ
- 大きなスコア値（20億以上）
- ユニーク識別子
- 長時間セッションでのフレームカウンター
- データベースキー

---

## Float

小数値とパーセンテージのための変数。

### 作成

```text
Create > Reactive SO > Variables > Float Variable
```

### 使用方法

```csharp
[SerializeField] private FloatVariableSO masterVolume;

// 書き込み
masterVolume.Value = 0.75f;

// 読み取り
audioSource.volume = masterVolume.Value;
```

### GPU Sync

```hlsl
// シェーダー内
float volume = _MasterVolume;
float healthFactor = saturate(_PlayerHealthPercent);
```

### 一般的なユースケース

- 音量設定
- ヘルス / スタミナ / マナ（0-1に正規化）
- 進行状況値（ローディング、クラフト）
- タイマーカウントダウン
- スピード倍率
- スポーン率

---

## Double

高精度計算のための倍精度変数。

### 作成

```text
Create > Reactive SO > Variables > Double Variable
```

### 使用方法

```csharp
[SerializeField] private DoubleVariableSO preciseDistance;

// 書き込み
preciseDistance.Value = Vector3.Distance(a, b);

// 読み取り
double distance = preciseDistance.Value;
```

### 一般的なユースケース

- 科学計算
- 金融値（高精度）
- 大規模座標システム
- 統計計算

---

## Bool

true/false状態のための変数。

### 作成

```text
Create > Reactive SO > Variables > Bool Variable
```

### 使用方法

```csharp
[SerializeField] private BoolVariableSO isPaused;

// 書き込み
isPaused.Value = true;

// 読み取り
if (isPaused.Value) Time.timeScale = 0;
```

### GPU Sync

```hlsl
// シェーダー内（0 = false, 1 = true）
int paused = _IsPaused;
if (paused) { /* ポーズエフェクトを適用 */ }
```

### 一般的なユースケース

- ゲームポーズ状態
- 機能トグル
- ミュート設定
- プレイヤー生存状態
- デバッグモード有効化
- チュートリアル完了

---

## String

名前、メッセージ、識別子のためのテキスト変数。

### 作成

```text
Create > Reactive SO > Variables > String Variable
```

### 使用方法

```csharp
[SerializeField] private StringVariableSO playerName;

// 書き込み
playerName.Value = "Hero";

// 読み取り
nameLabel.text = playerName.Value;
```

### 一般的なユースケース

- プレイヤー名
- 現在のレベル名
- ステータスメッセージ
- ローカライズキー
- セーブスロット名

---

## Vector2

2Dベクトル変数。位置と入力用。

### 作成

```text
Create > Reactive SO > Variables > Vector2 Variable
```

### 使用方法

```csharp
[SerializeField] private Vector2VariableSO inputAxis;

// 書き込み
inputAxis.Value = new Vector2(Input.GetAxis("Horizontal"), Input.GetAxis("Vertical"));

// 読み取り
Vector2 input = inputAxis.Value;
```

### GPU Sync

```hlsl
// シェーダー内
float4 input = _InputAxis;  // xyがVector2値を含む
float2 axis = input.xy;
```

### 一般的なユースケース

- 入力軸値
- 2D位置
- スクリーン位置
- ミニマップ座標
- ジョイスティック値

---

## Vector3

3D位置と方向のための変数。

### 作成

```text
Create > Reactive SO > Variables > Vector3 Variable
```

### 使用方法

```csharp
[SerializeField] private Vector3VariableSO playerPosition;

// 書き込み
playerPosition.Value = transform.position;

// 読み取り
Vector3 pos = playerPosition.Value;
```

### GPU Sync

```hlsl
// シェーダー内
float4 pos = _PlayerPosition;  // xyzがVector3値を含む
float3 playerPos = pos.xyz;
float distance = length(worldPos - playerPos);
```

### 一般的なユースケース

- プレイヤー位置（シェーダー用）
- ターゲット位置
- スポーンポイント
- 風の方向
- ライトの方向

---

## Quaternion

3D方向のための回転変数。

### 作成

```text
Create > Reactive SO > Variables > Quaternion Variable
```

### 使用方法

```csharp
[SerializeField] private QuaternionVariableSO cameraRotation;

// 書き込み
cameraRotation.Value = Camera.main.transform.rotation;

// 読み取り
Quaternion rot = cameraRotation.Value;
```

### GPU Sync

```hlsl
// シェーダー内
float4 rot = _CameraRotation;  // xyzwがクォータニオンコンポーネントを含む
```

### 一般的なユースケース

- カメラの方向
- オブジェクトの回転
- プロシージャル回転

---

## Color

テーマとビジュアルエフェクトのための色変数。

### 作成

```text
Create > Reactive SO > Variables > Color Variable
```

### 使用方法

```csharp
[SerializeField] private ColorVariableSO themeColor;

// 書き込み
themeColor.Value = Color.red;

// 読み取り
uiImage.color = themeColor.Value;
```

### GPU Sync

```hlsl
// シェーダー内
float4 theme = _ThemeColor;  // RGBA値
float3 rgb = theme.rgb;
```

### 一般的なユースケース

- UIテーマカラー
- 危険インジケーターカラー
- ヘルスバーティント
- 環境フォグカラー
- ライティングカラー

---

## GameObject

ターゲット追跡のためのオブジェクト参照変数。

### 作成

```text
Create > Reactive SO > Variables > GameObject Variable
```

### 使用方法

```csharp
[SerializeField] private GameObjectVariableSO currentTarget;

// 書き込み
currentTarget.Value = hitEnemy;

// 読み取り
if (currentTarget.Value != null)
{
    attackTarget = currentTarget.Value;
}
```

### 注意事項

- 「オブジェクトなし」を示すために`null`を保持可能
- オブジェクトが破棄されると参照が無効になる可能性あり
- 使用前に常にnullチェック
- GPU Syncサポートなし

### 一般的なユースケース

- 現在のターゲット参照
- 選択されたオブジェクト
- インタラクト中のオブジェクト
- 最も近い敵
- アクティブな武器

---

## カスタムタイプの作成

`VariableSO<T>`を継承してカスタム変数タイプを作成できます。

```csharp
using Tang3cko.ReactiveSO;
using UnityEngine;

[CreateAssetMenu(
    fileName = "WeaponVariable",
    menuName = "Reactive SO/Variables/Weapon Variable"
)]
public class WeaponVariableSO : VariableSO<WeaponData>
{
    // Value、InitialValue、ResetToInitial()が自動的に継承されます
}

[System.Serializable]
public struct WeaponData
{
    public string Name;
    public int Damage;
    public float Range;
}
```

変更通知用に対応するイベントチャンネルを作成します。

```csharp
[CreateAssetMenu(
    fileName = "WeaponEvent",
    menuName = "Reactive SO/Channels/Weapon Event"
)]
public class WeaponEventChannelSO : EventChannelSO<WeaponData>
{
}
```

カスタムタイプは`SyncValueToGPU()`をオーバーライドしない限りGPU Syncをサポートしません。

---

## 参考資料

- [変数ガイド]({{ '/ja/guides/variables' | relative_url }}) - 変数の使い方
- [イベントタイプリファレンス](event-types) - すべてのイベントチャンネルタイプ
- [デバッグ概要]({{ '/ja/debugging/' | relative_url }}) - 変数値のデバッグ
