---
layout: default
title: Asset-based DI
parent: Design Philosophy
nav_order: 2.5
---

# Asset-based Dependency Injection

---

## Purpose

This page introduces the concept of **Asset-based Dependency Injection (DI)**. You will learn why Reactive SO treats ScriptableObjects not just as data containers, but as the primary mechanism for resolving dependencies between systems.

---

## What is Dependency Injection?

Before diving into Asset-based DI, let's clarify what Dependency Injection actually means.

### The definition

[Martin Fowler defined Dependency Injection in 2004](https://martinfowler.com/articles/injection.html) as:

> "The basic idea of the Dependency Injection is to have a separate object, an assembler, that populates a field in the lister class with an appropriate implementation."

In simpler terms: **a component receives its dependencies from outside rather than creating them itself**.

### Three forms of DI

| Form | Description | Example |
|:-----|:------------|:--------|
| **Constructor Injection** | Dependencies passed via constructor | `new Service(dependency)` |
| **Setter Injection** | Dependencies assigned via setter methods | `service.SetDependency(dep)` |
| **Field Injection** | Dependencies assigned directly to fields | `service.dependency = dep` |

### DI ≠ DI Container

{: .important }
> **Dependency Injection** is a design pattern. **DI Containers** (like VContainer or Zenject) are tools that automate DI. You do not need a DI Container to practice Dependency Injection.

This distinction is crucial:

| Term | What it is |
|:-----|:-----------|
| **Dependency Injection** | The *pattern* of providing dependencies externally |
| **DI Container** | A *tool* that automates dependency resolution and lifecycle management |
| **Pure DI / Manual DI** | Practicing DI without a container, assembling dependencies by hand |

Many developers assume "DI = Zenject" or "DI = VContainer". This conflates the tool with the pattern. VContainer *performs* DI internally, but VContainer itself is not DI.

### Pure DI in Unity

When you write this code:

```csharp
public class PlayerController : MonoBehaviour
{
    [SerializeField] private IntVariableSO health;  // Dependency
}
```

And drag an asset into the Inspector field, you are practicing **Pure DI via Field Injection**. The Unity Inspector acts as the "assembler" that populates the dependency. No DI Container required.

---

## Beyond data containers

A common misconception is that ScriptableObjects are only for static data (like weapon stats or game settings). While they excel at that, their true power lies in their **lifecycle** and **identity**.

ScriptableObjects are:
1.  **Shared Instances**: They exist independently of scenes.
2.  **Asset-Referenced**: They have a unique GUID and can be assigned in the Inspector.

Reactive SO leverages these properties to use ScriptableObjects as **dynamic connection points** between systems.

---

## Comparison of Unity types

Each approach has its own strengths. Understanding these trade-offs helps you choose the right tool for each situation.

| Feature | ScriptableObject | Static Class (Singleton) | MonoBehaviour | Pure C# Class |
|:---|:---|:---|:---|:---|
| **Lifecycle** | Project-scoped<br>(Lives as long as referenced) | App Domain<br>(Reset on domain reload) | Scene-scoped<br>(Bound to scene lifecycle) | Managed by GC |
| **Injection** | Inspector (Asset)<br>Drag & Drop | Direct access<br>`Manager.Instance` | Inspector (Scene)<br>or `GetComponent` | Constructor / Factory |
| **Testability** | `CreateInstance` in tests | Requires reset between tests | Requires GameObject setup | Easy to instantiate |
| **Configuration** | Serialized<br>Edit in Inspector | Code or config files | Serialized<br>Per-instance in scene | Code only |
| **Multiplicity** | Multiple assets possible | Single global instance | Multiple GameObjects | Multiple instances |

### When to use each

| Approach | Best suited for |
|:---------|:----------------|
| **ScriptableObject** | Shared state across scenes, data-driven configuration, decoupled communication |
| **Static Class / Singleton** | Truly global services (e.g., logging), simple prototypes, quick access patterns |
| **MonoBehaviour** | Scene-specific behavior, GameObjects with lifecycle needs, visual components |
| **Pure C# Class** | Business logic, algorithms, domain models, maximum testability |

### Trade-offs of ScriptableObject

ScriptableObjects are not universally superior. Consider these trade-offs:

- **Runtime data persistence** - Non-serialized data can be lost if the asset is unloaded (see [Troubleshooting]({{ '/en/troubleshooting#data-loss' | relative_url }}))
- **Not scene-specific by design** - For per-scene state, MonoBehaviours may be more appropriate
- **Asset Management Hell** - This is real. A mature project using this pattern will have dozens or hundreds of assets: `OnPlayerDeath`, `OnEnemySpawned`, `PlayerHealth`, `EnemySet`... Finding the right asset, avoiding duplicates, and maintaining naming conventions becomes a genuine challenge. Reactive SO provides tools like [Asset Browser]({{ '/en/debugging/asset-browser' | relative_url }}) and [Dependency Analyzer]({{ '/en/debugging/dependency-analyzer' | relative_url }}) to help, but the overhead is unavoidable.

Reactive SO chooses ScriptableObjects because they excel at **cross-scene shared state** and **decoupled communication** - the specific problems this architecture addresses. But be aware: you are trading code complexity for asset complexity.

---

## The concept: Asset-based DI

**Asset-based Dependency Injection** is **Pure DI via Inspector**, where dependencies are resolved by referencing **Assets** instead of **Classes** or **Scene Objects**.

### Traditional (Tight Coupling)

```csharp
public class EnemySpawner : MonoBehaviour
{
    // Dependency on a specific Scene Object or Singleton
    void Spawn()
    {
        // "I need to find the specific Manager in this scene"
        GameManager.Instance.RegisterEnemy(newEnemy);
    }
}
```

This approach creates a hard dependency on `GameManager`. You cannot test `EnemySpawner` without `GameManager`, and you cannot have multiple enemy lists.

### Asset-based DI (Loose Coupling)

```csharp
public class EnemySpawner : MonoBehaviour
{
    // Dependency injected via Inspector (Field Injection)
    [SerializeField] private GameObjectRuntimeSetSO enemySet;

    void Spawn()
    {
        // "I don't care who holds this list, I just add to it"
        enemySet.Add(newEnemy);
    }
}
```

This is **Pure DI**: the dependency (`enemySet`) is provided externally via the Inspector. The Unity Inspector serves as the "assembler" in Fowler's terminology, manually wiring dependencies without requiring a DI Container framework.

---

## Case study: Runtime Sets as DI

[Runtime Sets]({{ '/en/guides/runtime-sets' | relative_url }}) are the most primitive and clear example of Asset-based DI.

1.  **The Problem**: Code needs a list of all enemies.
2.  **The Naive Solution**: Make a static `public static List<Enemy> AllEnemies` in a Manager.
    *   *Issue*: Hard to test, hard to manage multiple contexts, creates hidden dependencies.
3.  **The Asset-based Solution**: Create an `EnemySet` asset (ScriptableObject).
    *   Enemies register themselves to the asset: `set.Add(this)`.
    *   Systems read from the asset: `foreach (var e in set.Items)`.

The `EnemySet` asset becomes a **stable anchor** in your project. Scenes can load and unload, MonoBehaviours can be created and destroyed, but the **connection point (the Asset)** remains constant.

---

## Why Reactive SO focuses on this

Reactive SO is designed to maximize the benefits of Asset-based DI:

1.  **Event Channels** - Assets that act as method calls (`Action<T>`).
2.  **Variables** - Assets that act as shared fields (`T value`).
3.  **Reactive Entity Sets** - Assets that act as in-memory databases.

By moving these responsibilities to Assets, your MonoBehaviours become **stateless logic containers** that are easier to write, read, and test.
