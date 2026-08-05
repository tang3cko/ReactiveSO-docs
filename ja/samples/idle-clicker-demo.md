---
layout: default
title: Idle Clicker デモ
parent: サンプル
nav_order: 6
---

# Idle Clicker Demo

## 概要

**Event Channels / Variables / Runtime Sets / Delegate Objects (ActionSO)** を1つのゲームループに統合した短時間のアーケードランです。`ActionSO` を実際に使っている唯一のサンプルです。

盤面の図形を叩いて現金を稼ぎます。請求書が一定間隔で届き、未払いのまま残すと取り立て率が上がって、稼ぎがそのぶん目減りします。請求書を払うと、それに紐づいたパークが実行されます。スタミナは自然回復しないので、尽きた時点でランは終了します。

## 使用している機能

| 機能 | アセット / クラス | 説明 |
| :--- | :--- | :--- |
| Runtime Set | `ActiveTargets` (TargetRuntimeSetSO) | 的が生存中だけ自分を登録します |
| Variable | `Stamina`, `MaxStamina`, `Cash`, `CollectorRate` | ラン全体で共有される状態 |
| Event Channel | `OnStaminaChanged`, `OnCashChanged`, `OnCollectorRateChanged` | 上記 Variable が自分で発火する変更通知チャンネル |
| Event Channel | `OnSmashed`, `OnBillIssued`, `OnBillPaid`, `OnRunEnded` | ランの通知 |
| Action | `BankPayout` (`ActionSO<int>`) | 報酬に取り立て率を適用して入金します |
| Action | `DeepBreathPerk`, `WindfallPerk`, `NegotiatorPerk`, `SecondWindPerk` | 請求書の支払いで解放されるパーク |
| Data | `BillSO` アセット | 各請求書が、支払いで解放する `ActionSO` を保持します |

## このサンプルが示す2つの原則

### Variable は自分の変更チャンネルを持っている

`VariableSO<T>` は `EventChannelSO<T>` をシリアライズフィールドとして保持し、`Value` の setter がそれを発火します。したがって書き込み側は `Value` を代入するだけです。

```csharp
// 正しい: Variable 自身が OnCashChanged を発火する
cash.Value += net;

// 誤り: チャンネルが2回鳴る
cash.Value += net;
onCashChanged.RaiseEvent(cash.Value);
```

購読側はチャンネルアセットを購読し、初期値だけ `OnEnable` で Variable から読みます。

### インスタンス固有のデータは C# フィールドに置く

的の種別・残りヒット数・寿命はオブジェクトごとに異なるので、`Target` コンポーネントの素のフィールドです。共有されるのは「生存中の的の集合」だけなので、ScriptableObject にするのもそれだけです。

```csharp
public class Target : MonoBehaviour
{
    [SerializeField] private TargetRuntimeSetSO activeTargets;  // グローバル
    public int Hits { get; private set; }                       // インスタンス固有

    private void OnEnable() => activeTargets?.Add(this);
    private void OnDisable() => activeTargets?.Remove(this);
}
```

`Hits` を `IntVariableSO` にすると、画面上の的すべてが1つの値を共有してしまいます。

## パークはコードではなくアセット

ラン中の修飾値はすべて Variable なので、パークは「Variable にクランプ付きで加算するアクションアセット」になります。

| パーク | 構成 |
| :--- | :--- |
| Deep Breath | `SequenceActionSO`: MaxStamina を上げてから Stamina を回復 |
| Windfall | `Cash` に加算 |
| Negotiator | `CollectorRate` から減算(0でクランプ) |
| Second Wind | `Stamina` に加算 |

パークの追加はアセットを作って `BillSO` に割り当てるだけで、コードは書きません。

## 使い方

1. `IdleClicker` シーンを開く
2. Play Mode に入る
3. 左側の図形をクリックして割る。青は安価、緑は高報酬、紫は3ヒット必要、**赤は Decoy で、叩くと請求書が増える**
4. 何にも当たらなかった空振りでもスタミナを1消費する
5. 払える額になったら請求書の **PAY** をクリックし、パークが Variable を即座に書き換えるのを確認する
6. 請求書を放置して `COLLECTOR` が上がり、収入が目減りするのを確認する

## ユースケース

| ユースケース | 例 |
| :--- | :--- |
| データ駆動の報酬 | クエストやドロップの報酬をデータアセット上の `ActionSO` として記述する |
| 設定可能な修飾効果 | 効果ごとにコードを書かずに共有ステータスをクランプ付きで増減するバフ |
| 疎結合な HUD | 値を生成しているシステムを参照してはいけない表示 |
| アクティブオブジェクトの追跡 | 探索せずに自己登録するスポーン済みオブジェクト |
