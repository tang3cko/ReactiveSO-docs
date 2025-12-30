---
layout: default
title: Persistence
parent: Reactive Entity Sets
grand_parent: Guides
nav_order: 5
---

# Persistence

---

## Purpose

This page explains why ReactiveEntitySetSO data can be lost during scene transitions and how to prevent it using the Manager Scene pattern with ReactiveEntitySetHolder.

---

## The problem

Unity may unload ScriptableObject assets from memory when they are not referenced by any loaded scene. ReactiveEntitySetSO stores runtime data in non-serialized fields, which means this data is lost when the asset is unloaded and reloaded.

### Symptoms

- Entities registered before a scene transition disappear after returning
- `Count` returns 0 despite entities having been registered
- Data appears to persist only when the SO is selected in the Inspector (editor only)

### Why this happens

ScriptableObjects are data containers, not memory managers. When no scene object references a SO, Unity may unload it to free memory. Upon reload, non-serialized fields reset to their default values.

---

## Solution: Manager Scene pattern

The recommended solution is to use a Manager Scene that persists across scene transitions and holds references to your ReactiveEntitySetSO assets.

### What is a Manager Scene

A Manager Scene is an additively loaded scene that is never unloaded. It typically contains:

- Game managers (GameManager, AudioManager, etc.)
- Persistent UI elements
- References to ScriptableObject assets that need to persist

### Step 1: Create a Manager Scene

1. Create a new scene named `ManagerScene` (or similar)
2. Add a GameObject named `Managers`
3. Add your manager components to this GameObject

### Step 2: Add ReactiveEntitySetHolder

1. Create an empty GameObject named `RES References`
2. Add the `ReactiveEntitySetHolder` component
3. Add your ReactiveEntitySetSO assets to the references list

```text
ManagerScene
└── Managers
└── RES References
    └── ReactiveEntitySetHolder
        └── references: [EnemyEntitySet, PlayerEntitySet, ...]
```

### Step 3: Load Manager Scene on startup

In your game's entry point, load the Manager Scene additively:

```csharp
using UnityEngine;
using UnityEngine.SceneManagement;

public class GameBootstrap : MonoBehaviour
{
    [SerializeField] private string managerSceneName = "ManagerScene";

    private void Awake()
    {
        // Load Manager Scene if not already loaded
        if (!SceneManager.GetSceneByName(managerSceneName).isLoaded)
        {
            SceneManager.LoadScene(managerSceneName, LoadSceneMode.Additive);
        }
    }
}
```

### Step 4: Load game scenes additively

Load your game scenes without unloading the Manager Scene:

```csharp
// Load a new level
SceneManager.LoadScene("Level_01", LoadSceneMode.Additive);

// Unload the previous level (Manager Scene stays loaded)
SceneManager.UnloadSceneAsync("Level_00");
```

---

## Editor utilities

ReactiveEntitySetHolder includes editor buttons to help manage references:

| Button | Description |
|--------|-------------|
| **Find All in Project** | Searches the project for all ReactiveEntitySetSO assets and adds them to the list |
| **Clear Missing** | Removes null entries from the list (e.g., after deleting assets) |

---

## Best practices

1. **Use one Manager Scene** - Keep all persistent references in a single scene for clarity
2. **Add references explicitly** - While "Find All in Project" is convenient, explicitly adding assets makes dependencies clearer
3. **Don't use DontDestroyOnLoad** - The Manager Scene pattern is cleaner than marking individual objects as DontDestroyOnLoad

---

## Next steps

- [Events](events) - Subscribe to entity and set-level changes
- [Patterns](patterns) - Real-world usage patterns
