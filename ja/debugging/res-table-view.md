---
layout: default
title: RES Table View
parent: デバッグ
nav_order: 5
---

# RES Table View

{: .warning }
> ReactiveEntitySetSO は実験的機能です。APIは将来のバージョンで変更される可能性があります。

## 目的

このガイドでは、ReactiveEntitySet のエンティティデータをテーブル形式で表示・編集するための RES Table View の使用方法を説明します。Play Mode の一時停止中にスナップショットをキャプチャし、個々のエンティティの値を確認・編集する方法、およびデータの保存・復元機能の使い方を学びます。

---

## ウィンドウを開く

以下のいずれかの方法で開きます。

1. **メニューから**: Window > Reactive SO > RES Table View
2. **Inspectorから**: ReactiveEntitySetSO アセットを選択し、「Open Table View」ボタンをクリック

---

## 基本的な使い方

### スナップショットのキャプチャ

1. Play Mode に入る
2. ゲームを一時停止する（Pause ボタン）
3. Target RES で対象の ReactiveEntitySetSO を選択
4. 「Capture Snapshot」をクリック

キャプチャが成功すると、エンティティデータがテーブルに表示されます。

{: .note }
> スナップショットは一時停止中のみキャプチャできます。再開するとデータはクリアされます。

### データの編集

1. 編集したいセルをダブルクリック
2. 新しい値を入力
3. Enter で確定、Escape でキャンセル

編集した値は即座に ReactiveEntitySet に反映され、`OnDataChanged` イベントが発火します。

---

## ツールバー

| 要素 | 説明 |
|------|------|
| Target RES | 対象の ReactiveEntitySetSO を選択 |
| Capture Snapshot | 現在のエンティティデータをキャプチャ |
| Save | キャプチャしたデータをバイナリファイルに保存 |
| Load | バイナリファイルからデータを読み込み、完全に上書き |

---

## Save / Load 機能

### Save

キャプチャしたスナップショットデータを `.resdata` ファイルとして保存します。

- **有効条件**: キャプチャ済みデータがある場合
- **ファイル形式**: バイナリ（.resdata）
- **保存内容**: EntityID と TData 構造体の全フィールド

### Load

{: .warning }
> Load は現在のエンティティデータを完全に上書きします。この操作は元に戻せません。

保存した `.resdata` ファイルから復元し、ReactiveEntitySet のデータを完全に置き換えます。

- **有効条件**: 一時停止中 かつ RES が選択されている場合
- **動作**: `RestoreSnapshot()` を呼び出し、全エンティティを置換
- **警告**: ファイル選択後に確認ダイアログが表示されます

### バイナリフォーマット

```
[Header] - 20 bytes
├─ Magic: "RES\0" (4 bytes)
├─ Version: int32
├─ TypeHash: int32 (TData型のハッシュ)
├─ DataSize: int32 (TData構造体のサイズ)
└─ EntityCount: int32

[Data]
├─ EntityIds: int32[] (EntityCount 個)
└─ EntityData: byte[] (EntityCount × DataSize bytes)
```

---

## 列の説明

| 列 | 説明 |
|----|------|
| Entity ID | エンティティの一意識別子（読み取り専用） |
| (TData フィールド) | TData 構造体の各フィールドが動的に列として表示される |

### 対応する型

| 型 | 表示形式 | 編集 |
|----|----------|------|
| int, float, byte 等 | 数値 | ✓ |
| bool | True/False | ✓ (トグル) |
| Enum | 値名 | ✓ (ドロップダウン) |
| Vector2, Vector3 | (x, y, z) | × |
| Quaternion | オイラー角 | × |
| Color | (r, g, b, a) | × |

{: .note }
> Vector や Quaternion などの複合型は表示のみで、現在は編集できません。

---

## ユースケース

### エンティティ状態の検査

特定のエンティティの現在の値を確認し、問題の原因を特定します。Play Mode中に一時停止してスナップショットをキャプチャすることで、その時点での全エンティティのデータを一覧できます。

### ゲームロジックのテスト

値を直接編集してゲームロジックの挙動をテストします。例えば、エンティティのHPを0に設定して死亡処理が正しく動作するか確認できます。

### バグ再現データの保存

バグが発生した時点のエンティティ状態を `.resdata` ファイルとして保存し、後で復元してデバッグを続行します。

### 回帰テストのベースライン

特定の状態を保存し、テスト実行時にロードすることで、同じ条件での動作確認が可能になります。

---

## 制限事項

- **Play Mode 必須**: Edit Mode では動作しません
- **一時停止必須**: キャプチャ・編集・ロードは一時停止中のみ可能
- **型互換性**: 異なる TData 型のファイルをロードすると警告が表示されます
- **複合型の編集**: Vector, Quaternion, Color は編集できません
- **大量データ**: 10,000+ エンティティでも仮想化により動作しますが、初回キャプチャに時間がかかる場合があります

---

## 参照

- [Monitor Window]({{ '/ja/debugging/monitor' | relative_url }}) - リアルタイムイベント追跡
- [ReactiveEntitySet ガイド]({{ '/ja/guides/reactive-entity-sets/' | relative_url }}) - 基本的な使い方
- [スナップショットと永続化]({{ '/ja/guides/reactive-entity-sets/persistence' | relative_url }}) - CreateSnapshot/RestoreSnapshot の詳細
