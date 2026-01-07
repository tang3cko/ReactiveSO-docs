---
layout: default
title: Testing
parent: Guides
nav_order: 6
---

# Testing

{: .note }
> This guide covers unit testing patterns for Reactive SO components using Unity Test Framework (NUnit).

---

## Purpose

This guide explains how to write unit tests for Reactive SO components. You will learn testing patterns for Event Channels, Variables, Runtime Sets, and Reactive Entity Sets, along with dependency injection techniques for isolating external dependencies.

---

## Prerequisites

### Unity Test Framework

Unity Test Framework is included with Unity. Access the Test Runner via:

```text
Window > General > Test Runner
```

### Edit Mode vs Play Mode

| Mode | Use For | Speed |
|------|---------|-------|
| **Edit Mode** | ScriptableObject logic, pure C# classes | Fast |
| **Play Mode** | MonoBehaviour lifecycle, scene integration | Slower |

Most Reactive SO tests run in Edit Mode because ScriptableObjects can be instantiated without a scene.

---

## Testing Event Channels

### Basic pattern

Create ScriptableObject instances directly in tests:

```csharp
using NUnit.Framework;
using UnityEngine;
using Tang3cko.ReactiveSO;

public class IntEventChannelSOTests
{
    private IntEventChannelSO channel;

    [SetUp]
    public void Setup()
    {
        // Create instance without asset file
        channel = ScriptableObject.CreateInstance<IntEventChannelSO>();
    }

    [TearDown]
    public void Teardown()
    {
        // Must use DestroyImmediate in Edit Mode
        Object.DestroyImmediate(channel);
    }

    [Test]
    public void RaiseEvent_WithSubscriber_NotifiesSubscriberWithValue()
    {
        // Arrange
        int receivedValue = 0;
        channel.OnEventRaised += (value) => receivedValue = value;

        // Act
        channel.RaiseEvent(42);

        // Assert
        Assert.That(receivedValue, Is.EqualTo(42));
    }
}
```

### Testing multiple subscribers

```csharp
[Test]
public void RaiseEvent_WithMultipleSubscribers_NotifiesAll()
{
    // Arrange
    int notificationCount = 0;
    channel.OnEventRaised += (value) => notificationCount++;
    channel.OnEventRaised += (value) => notificationCount++;
    channel.OnEventRaised += (value) => notificationCount++;

    // Act
    channel.RaiseEvent(10);

    // Assert
    Assert.That(notificationCount, Is.EqualTo(3));
}
```

### Testing unsubscription

```csharp
[Test]
public void Unsubscribe_AfterRaiseEvent_DoesNotNotify()
{
    // Arrange
    bool wasNotified = false;
    System.Action<int> handler = (value) => wasNotified = true;
    channel.OnEventRaised += handler;
    channel.OnEventRaised -= handler;

    // Act
    channel.RaiseEvent(42);

    // Assert
    Assert.That(wasNotified, Is.False);
}
```

---

## Testing Variables

### Basic pattern

```csharp
public class IntVariableSOTests
{
    private IntVariableSO variable;
    private IntEventChannelSO eventChannel;

    [SetUp]
    public void Setup()
    {
        variable = ScriptableObject.CreateInstance<IntVariableSO>();
        eventChannel = ScriptableObject.CreateInstance<IntEventChannelSO>();
    }

    [TearDown]
    public void Teardown()
    {
        Object.DestroyImmediate(variable);
        Object.DestroyImmediate(eventChannel);
    }

    [Test]
    public void Value_Set_ChangesValue()
    {
        // Arrange
        variable.Value = 10;

        // Act
        variable.Value = 20;

        // Assert
        Assert.That(variable.Value, Is.EqualTo(20));
    }
}
```

### Testing change detection

Variables only fire events when the value actually changes:

```csharp
[Test]
public void Value_SetSameValue_DoesNotRaiseEvent()
{
    // Arrange
    variable.Value = 42;
    bool eventRaised = false;

    // Set up event channel using reflection (internal field)
    var field = typeof(VariableSO<int>).GetField("onValueChanged",
        System.Reflection.BindingFlags.NonPublic |
        System.Reflection.BindingFlags.Instance);
    field.SetValue(variable, eventChannel);

    eventChannel.OnEventRaised += (value) => eventRaised = true;

    // Act
    variable.Value = 42;  // Same value

    // Assert
    Assert.That(eventRaised, Is.False);
}
```

---

## Testing Runtime Sets

### Basic pattern

```csharp
public class GameObjectRuntimeSetSOTests
{
    private GameObjectRuntimeSetSO runtimeSet;
    private GameObject testObject;

    [SetUp]
    public void Setup()
    {
        runtimeSet = ScriptableObject.CreateInstance<GameObjectRuntimeSetSO>();
        testObject = new GameObject("TestObject");
    }

    [TearDown]
    public void Teardown()
    {
        Object.DestroyImmediate(testObject);
        Object.DestroyImmediate(runtimeSet);
    }

    [Test]
    public void Add_ValidObject_IncreasesCount()
    {
        // Arrange
        Assert.That(runtimeSet.Count, Is.EqualTo(0));

        // Act
        runtimeSet.Add(testObject);

        // Assert
        Assert.That(runtimeSet.Count, Is.EqualTo(1));
    }
}
```

---

## Testing Reactive Entity Sets

### Test-specific subclass pattern

Create a concrete implementation for testing generic classes:

```csharp
public class ReactiveEntitySetSOTests
{
    // Test data structure
    public struct TestEntityData
    {
        public int Health;
        public float Speed;
    }

    // Test implementation
    private class TestReactiveEntitySetSO : ReactiveEntitySetSO<TestEntityData>
    {
    }

    // Test owner component
    private class TestOwner : MonoBehaviour { }

    private TestReactiveEntitySetSO entitySet;
    private GameObject ownerObject;
    private TestOwner owner;

    [SetUp]
    public void Setup()
    {
        entitySet = ScriptableObject.CreateInstance<TestReactiveEntitySetSO>();
        ownerObject = new GameObject("Owner");
        owner = ownerObject.AddComponent<TestOwner>();
    }

    [TearDown]
    public void Teardown()
    {
        Object.DestroyImmediate(ownerObject);
        Object.DestroyImmediate(entitySet);
    }

    [Test]
    public void Register_ValidOwner_AddsToSet()
    {
        // Arrange
        var data = new TestEntityData { Health = 100, Speed = 5f };

        // Act
        entitySet.Register(owner, data);

        // Assert
        Assert.That(entitySet.Count, Is.EqualTo(1));
        Assert.That(entitySet.Contains(owner), Is.True);
    }

    [Test]
    public void UpdateData_ModifiesState()
    {
        // Arrange
        entitySet.Register(owner, new TestEntityData { Health = 100 });

        // Act
        entitySet.UpdateData(owner, state => {
            state.Health -= 30;
            return state;
        });

        // Assert
        var data = entitySet.GetData(owner);
        Assert.That(data.Health, Is.EqualTo(70));
    }
}
```

---

## Dependency Injection pattern

For classes with external dependencies (file I/O, dialogs), use interfaces and manual mocks.

### Define an interface

```csharp
public interface IFileService
{
    string SaveFilePanel(string title, string directory,
        string defaultName, string extension);
    void WriteAllText(string path, string content, Encoding encoding);
    void RevealInFinder(string path);
}
```

### Create a mock implementation

```csharp
public class MockFileService : IFileService
{
    public string PathToReturn { get; set; }
    public bool WriteAllTextCalled { get; private set; }
    public string LastWrittenPath { get; private set; }
    public string LastWrittenContent { get; private set; }

    public string SaveFilePanel(string title, string directory,
        string defaultName, string extension)
    {
        return PathToReturn;
    }

    public void WriteAllText(string path, string content, Encoding encoding)
    {
        WriteAllTextCalled = true;
        LastWrittenPath = path;
        LastWrittenContent = content;
    }

    public void RevealInFinder(string path) { }
}
```

### Use in tests

```csharp
public class ExporterTests
{
    private MockFileService mockFileService;
    private MockDialogService mockDialogService;
    private Exporter exporter;

    [SetUp]
    public void Setup()
    {
        mockFileService = new MockFileService();
        mockDialogService = new MockDialogService();
        exporter = new Exporter(mockDialogService, mockFileService);
    }

    [Test]
    public void Export_WithValidPath_WritesFile()
    {
        // Arrange
        mockFileService.PathToReturn = "/path/to/file.csv";

        // Act
        exporter.Export(testData);

        // Assert
        Assert.That(mockFileService.WriteAllTextCalled, Is.True);
        Assert.That(mockFileService.LastWrittenPath,
            Is.EqualTo("/path/to/file.csv"));
    }

    [Test]
    public void Export_UserCancels_DoesNotWriteFile()
    {
        // Arrange
        mockFileService.PathToReturn = "";  // User cancelled

        // Act
        exporter.Export(testData);

        // Assert
        Assert.That(mockFileService.WriteAllTextCalled, Is.False);
    }
}
```

---

## Best practices

### Follow AAA pattern

Structure tests with clear sections:

```csharp
[Test]
public void MethodName_Scenario_ExpectedBehavior()
{
    // Arrange - Set up test data and conditions

    // Act - Execute the code under test

    // Assert - Verify the results
}
```

### Always clean up

Use `DestroyImmediate` in Edit Mode tests:

```csharp
[TearDown]
public void Teardown()
{
    // Destroy does not work immediately in Edit Mode
    Object.DestroyImmediate(channel);
}
```

### Test edge cases

```csharp
[Test]
public void RaiseEvent_WithoutSubscribers_DoesNotThrow()
{
    Assert.DoesNotThrow(() => channel.RaiseEvent(42));
}

[Test]
public void RaiseEvent_WithMinValue_TransmitsMinValue()
{
    int receivedValue = 0;
    channel.OnEventRaised += (value) => receivedValue = value;

    channel.RaiseEvent(int.MinValue);

    Assert.That(receivedValue, Is.EqualTo(int.MinValue));
}
```

### Keep tests independent

Each test should be able to run in isolation:

```csharp
// BAD: Tests share state
private static IntEventChannelSO sharedChannel;

// GOOD: Each test creates its own instance
[SetUp]
public void Setup()
{
    channel = ScriptableObject.CreateInstance<IntEventChannelSO>();
}
```

---

## Common pitfalls

### Forgetting to destroy ScriptableObjects

ScriptableObjects created with `CreateInstance` persist until destroyed. Always clean up in `TearDown`.

### Using `new` instead of `CreateInstance`

```csharp
// BAD: Will throw error
var channel = new IntEventChannelSO();

// GOOD: Correct way to create ScriptableObjects
var channel = ScriptableObject.CreateInstance<IntEventChannelSO>();
```

### Testing in Play Mode unnecessarily

Most Reactive SO logic can be tested in Edit Mode, which is much faster.

---

## Summary

| Component | Key Pattern |
|-----------|-------------|
| Event Channels | CreateInstance, subscribe, verify callback |
| Variables | CreateInstance, set value, check change detection |
| Runtime Sets | Create GameObject, add/remove, verify count |
| Reactive Entity Sets | Test subclass, Register/UpdateData, verify state |
| External Dependencies | Interface + Mock pattern |

---

## References

- [Testability]({{ '/en/design-philosophy/testability' | relative_url }}) - Why Reactive SO is naturally testable
- [Unity Test Framework](https://docs.unity3d.com/Packages/com.unity.test-framework@latest) - Official documentation
- [NUnit Documentation](https://docs.nunit.org/) - Test framework reference
