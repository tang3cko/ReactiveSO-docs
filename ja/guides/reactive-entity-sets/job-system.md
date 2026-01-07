---
layout: default
title: Job System連携
parent: Reactive Entity Sets
grand_parent: ガイド
nav_order: 6
---

# Job System連携

---

## 目的

このガイドでは、ReactiveEntitySetをUnityのJob Systemと連携させて高性能な並列処理を行う方法を説明します。Orchestratorパターンをいつ使うべきか、ダブルバッファリングの仕組み、数千のエンティティを効率的にシミュレーションする実装方法を学びます。

---

## いつJob System連携を使うか

### Orchestratorを使う場合

- **1,000以上のエンティティ**を毎フレーム更新する
- パフォーマンスのために**Burstコンパイル**が必要
- **物理シミュレーション**や**AI計算**を実行する
- エンティティ更新をCPUコア間で**並列化**したい

### 直接SetDataを使う場合

- エンティティが**1,000未満**
- 更新が**毎フレームではない**（頻度が低い）
- **即座の**イベント通知が必要
- パフォーマンスよりシンプルさが重要

### パフォーマンス比較

| アプローチ | 1,000エンティティ | 10,000エンティティ | 100,000エンティティ |
|----------|------------------|-------------------|---------------------|
| 直接SetData | ~0.5ms | ~5ms | ~50ms |
| Orchestrator + Jobs | ~0.1ms | ~0.3ms | ~2ms |

{: .note }
> これらの数値は概算であり、データ構造のサイズと更新ロジックの複雑さによって異なります。

---

## コアコンセプト

### ダブルバッファリング

Orchestratorはデータの2つのコピーを保持します。

```
フレームN:
┌─────────────────┐     ┌─────────────────┐
│  フロントバッファ  │     │  バックバッファ   │
│  (現在の状態)     │     │  (次の状態)      │
│                 │     │                 │
│  【読み取り】     │     │  【書き込み】     │
│   レンダリング    │     │   Jobs          │
│   UI            │     │   シミュレーション  │
└─────────────────┘     └─────────────────┘

フレームN+1 (CompleteAndApply後):
┌─────────────────┐     ┌─────────────────┐
│  バックバッファ   │     │  フロントバッファ  │
│  (今は現在)      │     │  (今は次)        │
│                 │     │                 │
│  バッファが      │ ←→  │  O(1)の         │
│  瞬時にスワップ   │     │  ポインタスワップ  │
└─────────────────┘     └─────────────────┘
```

このパターンにより以下が可能になります。

- **ロックフリーの読み取り**: メインスレッドがフロントバッファを読み取る間、Jobsがバックバッファに書き込み
- **コピー不要**: バッファはポインタ交換でスワップし、データコピーは発生しない
- **一貫した状態**: 読み取り側は常に完全で一貫したフレームを参照

### Orchestratorのライフサイクル

```csharp
void Start()
{
    // 1. Orchestratorを作成（EntitySetをラップ）
    orchestrator = new ReactiveEntitySetOrchestrator<MyData>(entitySet);
}

void Update()
{
    // 2. バックバッファに書き込むJobsをスケジュール
    var handle = ScheduleMyJob();
    orchestrator.ScheduleUpdate(handle, entityCount);
}

void LateUpdate()
{
    // 3. Jobsを完了してバッファをスワップ
    orchestrator.CompleteAndApply();
}

void OnDestroy()
{
    // 4. ネイティブメモリをクリーンアップ
    orchestrator?.Dispose();
}
```

---

## ステップバイステップ実装

### ステップ1: データ構造体を定義

Job System互換性のため、データは`unmanaged`（マネージド参照なし）である必要があります。

```csharp
// Good: すべてのフィールドがunmanaged型
public struct UnitState
{
    public float2 position;
    public float2 velocity;
    public int health;
    public int teamId;
}

// Bad: マネージド参照を含む
public struct BadState
{
    public string name;      // stringはマネージド - コンパイルエラー
    public GameObject target; // 参照型 - コンパイルエラー
}
```

{: .warning }
> `unmanaged`制約は`ReactiveEntitySetSO<TData>`によって強制されます。構造体にマネージド型が含まれている場合、コンパイルエラーになります。

### ステップ2: EntitySetアセットを作成

```csharp
using Tang3cko.ReactiveSO;
using UnityEngine;

[CreateAssetMenu(
    fileName = "UnitStateSet",
    menuName = "Reactive SO/Entity Sets/Unit State"
)]
public class UnitStateSetSO : ReactiveEntitySetSO<UnitState>
{
    // ベースクラスがすべての機能を提供
}
```

### ステップ3: シミュレーションマネージャーを作成

```csharp
using UnityEngine;
using Unity.Collections;
using Unity.Jobs;
using Tang3cko.ReactiveSO;

public class SimulationManager : MonoBehaviour
{
    [SerializeField] private UnitStateSetSO unitSet;

    private ReactiveEntitySetOrchestrator<UnitState> orchestrator;

    private void Start()
    {
        // ダブルバッファリングを管理するOrchestratorを作成
        orchestrator = new ReactiveEntitySetOrchestrator<UnitState>(unitSet);

        // 初期ユニットをスポーン
        SpawnUnits();
    }

    private void Update()
    {
        if (unitSet.Count == 0) return;

        // シミュレーションJobをスケジュール
        JobHandle handle = ScheduleSimulation();

        // OrchestratorにペンディングJobを通知
        orchestrator.ScheduleUpdate(handle, unitSet.Count);
    }

    private void LateUpdate()
    {
        // Jobを完了してバッファをスワップ
        // これにより新しい状態がEntitySetに反映される
        orchestrator?.CompleteAndApply();
    }

    private void OnDestroy()
    {
        // ネイティブメモリを解放するため必ずDispose
        orchestrator?.Dispose();
        orchestrator = null;
    }

    private void SpawnUnits()
    {
        for (int i = 0; i < 10000; i++)
        {
            unitSet.Register(i, new UnitState
            {
                position = Random.insideUnitCircle * 100f,
                velocity = Random.insideUnitCircle,
                health = 100,
                teamId = i % 4
            });
        }
    }

    private JobHandle ScheduleSimulation()
    {
        // フロントバッファから読み取り（現在の状態）
        NativeSlice<UnitState> srcData = unitSet.Data;
        NativeSlice<int> srcIds = unitSet.EntityIds;

        // バックバッファに書き込み（次の状態）
        NativeArray<UnitState> dstData = orchestrator.GetBackBuffer();
        NativeArray<int> dstIds = orchestrator.GetBackBufferIds();

        // IDをコピー（このシミュレーションでは変更なし）
        new NativeSlice<int>(dstIds, 0, srcIds.Length).CopyFrom(srcIds);

        // シミュレーションJobをスケジュール
        var job = new UnitSimulationJob
        {
            Input = srcData,
            Output = dstData,
            DeltaTime = Time.deltaTime
        };

        return job.Schedule(srcData.Length, 64);
    }
}
```

### ステップ4: BurstコンパイルされたJobを作成

```csharp
using Unity.Burst;
using Unity.Collections;
using Unity.Jobs;
using Unity.Mathematics;

[BurstCompile]
public struct UnitSimulationJob : IJobParallelFor
{
    [ReadOnly] public NativeSlice<UnitState> Input;
    public NativeArray<UnitState> Output;
    public float DeltaTime;

    public void Execute(int index)
    {
        UnitState unit = Input[index];

        // 速度に基づいて位置を更新
        unit.position += unit.velocity * DeltaTime;

        // 境界ラッピング
        unit.position = math.fmod(unit.position + 100f, 200f) - 100f;

        // 出力バッファに書き込み
        Output[index] = unit;
    }
}
```

---

## 高度なパターン

### パターン1: Jobでマップデータを読み取る

Jobsはエンティティデータと一緒に他のNativeArrayから読み取ることができます。

```csharp
[BurstCompile]
public struct TerrainAwareJob : IJobParallelFor
{
    [ReadOnly] public NativeSlice<UnitState> Units;
    [ReadOnly] public NativeArray<int> TerrainMap;
    public NativeArray<UnitState> Results;
    public int MapWidth;
    public int MapHeight;

    public void Execute(int index)
    {
        var unit = Units[index];

        // ユニット位置で地形をサンプリング
        int x = (int)math.clamp(unit.position.x, 0, MapWidth - 1);
        int y = (int)math.clamp(unit.position.y, 0, MapHeight - 1);
        int terrain = TerrainMap[y * MapWidth + x];

        // 地形に基づいて動作を変更
        if (terrain == 0) // 水
        {
            unit.velocity *= 0.5f; // 減速
        }

        Results[index] = unit;
    }
}
```

### パターン2: 共有データへの書き込み（注意が必要）

複数のJobsが共有データに書き込む必要がある場合は`[NativeDisableParallelForRestriction]`を使用します。

```csharp
[BurstCompile]
public struct TerritoryPaintJob : IJobParallelFor
{
    [ReadOnly] public NativeSlice<UnitState> Units;
    public NativeArray<UnitState> Results;

    [NativeDisableParallelForRestriction]
    public NativeArray<int> TerritoryMap; // 共有書き込み先

    public int MapWidth;

    public void Execute(int index)
    {
        var unit = Units[index];

        // ユニット位置に領土をペイント
        int x = (int)unit.position.x;
        int y = (int)unit.position.y;
        int mapIndex = y * MapWidth + x;

        // 注意: 複数ユニットが同じセルに書き込む可能性あり
        // 最後の書き込みが勝つ（領土ペイントでは許容可能）
        TerritoryMap[mapIndex] = unit.teamId;

        Results[index] = unit;
    }
}
```

{: .warning }
> 複数スレッドが同じメモリ位置に書き込む場合、結果は非決定的です。「最後の書き込みが勝つ」動作が許容できる場合にのみこのパターンを使用してください。

### パターン3: 複数のJobsを連鎖

```csharp
private JobHandle ScheduleSimulation()
{
    var srcData = unitSet.Data;
    var dstData = orchestrator.GetBackBuffer();

    // Job 1: 物理更新
    var physicsJob = new PhysicsJob
    {
        Input = srcData,
        Output = dstData,
        DeltaTime = Time.deltaTime
    };
    JobHandle physicsHandle = physicsJob.Schedule(srcData.Length, 64);

    // Job 2: AI更新（物理に依存）
    var aiJob = new AIJob
    {
        Data = dstData, // 物理出力から読み取り
        Targets = targetPositions
    };
    JobHandle aiHandle = aiJob.Schedule(srcData.Length, 64, physicsHandle);

    // Job 3: 衝突検出（AIに依存）
    var collisionJob = new CollisionJob
    {
        Data = dstData
    };
    JobHandle collisionHandle = collisionJob.Schedule(srcData.Length, 64, aiHandle);

    return collisionHandle;
}
```

---

## Snapshot API

Snapshot APIを使用すると、EntitySetの完全な状態をキャプチャして復元できます。

### スナップショットの作成

```csharp
// 現在の状態をキャプチャ
EntitySetSnapshot<UnitState> snapshot = unitSet.CreateSnapshot(Allocator.Persistent);

// snapshot.Data にはすべてのエンティティデータのコピーが含まれる
// snapshot.EntityIds にはすべてのエンティティIDのコピーが含まれる
// snapshot.Count にはエンティティ数が含まれる
```

### スナップショットの復元

```csharp
// 以前の状態に復元
unitSet.RestoreSnapshot(snapshot);
```

復元により以下の処理が行われます。

1. 現在のEntitySetがクリアされる
2. スナップショットからすべてのエンティティが登録される
3. OnSetChangedイベントが発火する

### メモリ管理

```csharp
// Persistentで割り当てたスナップショットはDisposeが必要
snapshot.Dispose();

// または短命なスナップショットにはAllocator.Tempを使用
using (var tempSnapshot = unitSet.CreateSnapshot(Allocator.Temp))
{
    // スナップショットを使用...
} // 自動的にDispose
```

### ユースケース: タイムトラベル / 巻き戻し

```csharp
public class TimeController : MonoBehaviour
{
    [SerializeField] private UnitStateSetSO unitSet;

    private List<EntitySetSnapshot<UnitState>> history = new();
    private int maxHistory = 300; // 60fpsで5秒

    private void LateUpdate()
    {
        // 毎フレーム履歴を記録
        if (history.Count >= maxHistory)
        {
            history[0].Dispose();
            history.RemoveAt(0);
        }
        history.Add(unitSet.CreateSnapshot(Allocator.Persistent));
    }

    public void Rewind(int frames)
    {
        int targetIndex = history.Count - 1 - frames;
        if (targetIndex >= 0 && targetIndex < history.Count)
        {
            unitSet.RestoreSnapshot(history[targetIndex]);
        }
    }

    private void OnDestroy()
    {
        foreach (var snapshot in history)
        {
            snapshot.Dispose();
        }
        history.Clear();
    }
}
```

### ユースケース: セーブ/ロード

```csharp
public void SaveGame(string path)
{
    var snapshot = unitSet.CreateSnapshot(Allocator.Temp);

    // シリアライズ用にバイト配列に変換
    byte[] dataBytes = new byte[snapshot.Count * UnsafeUtility.SizeOf<UnitState>()];
    byte[] idBytes = new byte[snapshot.Count * sizeof(int)];

    snapshot.Data.Slice(0, snapshot.Count).CopyTo(
        NativeArrayUnsafeUtility.ConvertExistingDataToNativeArray<UnitState>(
            dataBytes, snapshot.Count, Allocator.None));

    // ファイルに保存...

    snapshot.Dispose();
}
```

---

## パフォーマンスのヒント

### 1. バッチサイズの調整

`Schedule()`の2番目のパラメータはバッチサイズです。ワークロードに基づいて調整してください。

```csharp
// 小さく高速な操作: 大きなバッチ
job.Schedule(count, 256);

// 複雑な操作: 小さなバッチ
job.Schedule(count, 32);

// 経験則: 64から始めてプロファイル
job.Schedule(count, 64);
```

### 2. メインスレッドの作業を最小化

```csharp
// Bad: メインスレッドでJobを待機
void Update()
{
    var handle = ScheduleJob();
    handle.Complete(); // メインスレッドをブロック！
    ProcessResults();
}

// Good: メインスレッドが他の作業をしている間Jobを実行
void Update()
{
    var handle = ScheduleJob();
    orchestrator.ScheduleUpdate(handle, count);
    // メインスレッドはレンダリング、入力などを継続
}

void LateUpdate()
{
    orchestrator.CompleteAndApply(); // フレーム終了時に完了
}
```

### 3. Job内でのアロケーションを避ける

```csharp
// Bad: Job内でNativeArrayを作成
public void Execute(int index)
{
    var temp = new NativeArray<float>(10, Allocator.Temp); // アロケーション！
}

// Good: 事前に割り当ててパラメータとして渡す
[NativeDisableParallelForRestriction]
public NativeArray<float> TempBuffer; // 事前割り当て済み
```

### 4. Burstを使用

10〜100倍のパフォーマンス向上のため、常にJobsに`[BurstCompile]`を追加してください。

```csharp
[BurstCompile]
public struct MyJob : IJobParallelFor
{
    // Burstがこれを高度に最適化されたネイティブコードにコンパイル
}
```

---

## よくある落とし穴

### 1. OrchestratorのDisposeを忘れる

```csharp
// メモリリーク！
void OnDestroy()
{
    // orchestrator.Dispose(); // これを忘れた！
}

// 正しい
void OnDestroy()
{
    orchestrator?.Dispose();
    orchestrator = null;
}
```

### 2. Job実行中にEntitySetから読み取る

```csharp
// 危険: Jobの実行中に読み取り
void Update()
{
    orchestrator.ScheduleUpdate(handle, count);

    // これはフロントバッファを読むので安全
    var state = unitSet[someId]; // OK

    // しかし変更はイベントを発火し問題を起こす可能性あり
    unitSet[someId] = newState; // Job実行中は避ける
}
```

### 3. IDのコピーを忘れる

```csharp
// バグ: IDがバックバッファにコピーされていない
private JobHandle ScheduleSimulation()
{
    var dstData = orchestrator.GetBackBuffer();
    var dstIds = orchestrator.GetBackBufferIds();

    // IDのコピーを忘れた！
    // new NativeSlice<int>(dstIds, 0, srcIds.Length).CopyFrom(srcIds);

    var job = new MyJob { Output = dstData };
    return job.Schedule(count, 64);
}
```

### 4. スナップショットに間違ったAllocatorを使用

```csharp
// バグ: 長命なスナップショットにTempアロケータを使用
var snapshot = unitSet.CreateSnapshot(Allocator.Temp);
history.Add(snapshot); // 4フレーム後に無効になる！

// 正しい: 長命なデータにはPersistentを使用
var snapshot = unitSet.CreateSnapshot(Allocator.Persistent);
history.Add(snapshot);
// 終了時にDisposeを忘れずに！
```

---

## 完全な例

完全な動作例については、パッケージ内の**TinyHistoryDemo**サンプルを参照してください。

```
Assets/_Project/Samples/TinyHistoryDemo/
├── Scripts/
│   ├── SimulationManager.cs    // Orchestratorの使用法
│   └── MapManager.cs           // テクスチャレンダリング
└── Scenes/
    └── TinyHistoryDemo.unity
```

このデモは10,000ユニットがマップに領土をペイントするシミュレーションで、最新ハードウェアで200fps以上を達成します。

---

## 参考資料

- [Job System連携（技術詳細）]({{ '/ja/design-philosophy/reactive-entity-sets/job-system-integration' | relative_url }}) - 詳細なアーキテクチャ
- [Unity Job Systemマニュアル](https://docs.unity3d.com/ja/current/Manual/JobSystem.html)
- [Burstコンパイラマニュアル](https://docs.unity3d.com/Packages/com.unity.burst@latest)
- [NativeArrayドキュメント](https://docs.unity3d.com/ScriptReference/Unity.Collections.NativeArray_1.html)
