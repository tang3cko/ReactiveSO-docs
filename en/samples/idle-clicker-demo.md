---
layout: default
title: Idle Clicker Demo
parent: Samples
nav_order: 6
---

# Idle Clicker Demo

## Overview

A short arcade run that integrates **Event Channels, Variables, Runtime Sets, and Delegate
Objects (ActionSO)** in one game loop. It is the only sample that exercises `ActionSO`.

Smash shapes on the board for cash. Bills arrive on a timer, and every unpaid bill raises the
collector's cut, which is taken out of everything you earn. Paying a bill runs the perk attached
to it. Stamina never regenerates, so the run ends when it reaches zero.

## Features Used

| Feature | Asset / Class | Description |
| :--- | :--- | :--- |
| Runtime Set | `ActiveTargets` (TargetRuntimeSetSO) | Targets register themselves while alive |
| Variable | `Stamina`, `MaxStamina`, `Cash`, `CollectorRate` | Global run state |
| Event Channel | `OnStaminaChanged`, `OnCashChanged`, `OnCollectorRateChanged` | Paired change channels owned by the Variables |
| Event Channel | `OnSmashed`, `OnBillIssued`, `OnBillPaid`, `OnRunEnded` | Run notifications |
| Action | `BankPayout` (`ActionSO<int>`) | Applies the collector's cut to a payout, then banks it |
| Action | `DeepBreathPerk`, `WindfallPerk`, `NegotiatorPerk`, `SecondWindPerk` | Perks unlocked by paying bills |
| Data | `BillSO` assets | Each bill owns the `ActionSO` paying it unlocks |

## The two rules this sample demonstrates

### A Variable owns its change channel

`VariableSO<T>` holds a serialized `EventChannelSO<T>`, and the `Value` setter raises it. A
writer therefore only assigns `Value`:

```csharp
// Correct: the Variable raises OnCashChanged by itself
cash.Value += net;

// Wrong: fires the channel twice
cash.Value += net;
onCashChanged.RaiseEvent(cash.Value);
```

Consumers subscribe to the channel asset and bootstrap the initial value from the Variable in
`OnEnable`.

### Instance data stays in C# fields

A target's kind, remaining hits, and lifetime are unique per object, so they are plain fields on
the `Target` component. Only the *set* of live targets is shared, so only that is a
ScriptableObject:

```csharp
public class Target : MonoBehaviour
{
    [SerializeField] private TargetRuntimeSetSO activeTargets;  // global
    public int Hits { get; private set; }                       // per instance

    private void OnEnable() => activeTargets?.Add(this);
    private void OnDisable() => activeTargets?.Remove(this);
}
```

Putting `Hits` in an `IntVariableSO` would make every target on screen share one value.

## Perks are assets, not code

Every run modifier is a Variable, so a perk is an action asset that adds a clamped delta to one:

| Perk | Composition |
| :--- | :--- |
| Deep Breath | `SequenceActionSO`: raise MaxStamina, then restore Stamina |
| Windfall | Add to `Cash` |
| Negotiator | Subtract from `CollectorRate`, clamped at 0 |
| Second Wind | Add to `Stamina` |

Adding a perk means authoring an asset and dropping it on a `BillSO`. No new code.

## How to Use

1. Open the `IdleClicker` scene
2. Enter Play Mode
3. Click a shape on the left to smash it. Blue is cheap, green pays more, purple takes three
   hits, **red is a Decoy - hitting it issues another bill**
4. A swing that hits nothing still costs 1 stamina
5. Click **PAY** on a bill once you can afford it, and watch its perk change a Variable
6. Leave a bill unpaid and watch `COLLECTOR` rise and your income shrink

## Use Cases

| Use Case | Example |
| :--- | :--- |
| Data-driven rewards | Quest or loot rewards authored as `ActionSO` assets on a data asset |
| Configurable modifiers | Buffs that clamp a shared stat without new code per effect |
| Decoupled HUD | Readouts that must not reference the system producing the value |
| Active object tracking | Spawned objects that register themselves instead of being searched for |
