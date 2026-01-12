---
layout: default
title: Tiny History Demo
parent: Samples
nav_order: 4
has_children: true
---

# Tiny History Demo

---

## Purpose

This sample demonstrates how Event Channels, Reactive Entity Sets, and GPU Sync work together in a production-style application. Use it as a reference for building complex systems with Reactive SO.

> **Advanced Showcase** - This is an integrated example showing all features working together. If you're new to Reactive SO, start with the simpler samples ([Basic Demo]({{ '/en/samples/basic-demo' | relative_url }}), [Mote Demo]({{ '/en/samples/mote-demo' | relative_url }})) and the feature guides first.
{: .note }

---

## What is Tiny History?

<!-- TODO: Add video/GIF showing simulation running and timeline seek in action -->

A nation simulation with timeline navigation. Watch AI-controlled nations compete for territory on a procedurally generated map, then rewind time to explore alternate histories.

- Nations expand territory through military conquest
- Armies march, battle, and capture provinces
- Timeline slider allows seeking to any point in history
- Event log chronicles major historical events

---

## Features Demonstrated

| Feature | Usage | Description |
| :--- | :--- | :--- |
| Event Channels | Domain events | Province ownership changes, nation elimination |
| Event Channels | UI communication | Timeline seek requests |
| Reactive Variables | State sync | Play/pause, current frame, simulation speed |
| Reactive Entity Sets | Jobs integration | Army, province, nation state with double buffering |
| Reactive Entity Sets | GPU Sync | Province ownership to map shader |
| GPU Sync | Rendering | Province colors, army positions |

---

## Getting Started

### Import the Sample

1. Open **Window > Package Manager**
2. Select **Reactive SO** in the list
3. Expand the **Samples** section
4. Click **Import** next to **Tiny History Demo**

### Open the Scene

Navigate to the imported sample folder and open `Scenes/TinyHistoryDemo.unity`.

### Play

Enter Play Mode. Nations will begin expanding and battling automatically.

---

## UI Controls

<!-- TODO: Add screenshot showing the UI controls (timeline slider, play/pause, speed slider, event log) -->

### Timeline Slider

Drag to seek to any point in recorded history. The simulation pauses when you seek.

### Play / Pause Button

Toggle simulation playback. When paused, you can seek through history without advancing.

### Speed Slider

Adjust simulation speed from 0.5x to 3x. Higher speeds progress history faster.

### Event Log (Chronicle)

View historical events as they occur:

- **Province captures**: Which nation took which territory
- **Nation eliminations**: When a nation loses all territory

Toggle between "Key" (eliminations only) and "All" (all events) filters.

### Status Panel

Displays current simulation state:

- Current year
- Living nations count
- Active armies count
- History snapshots stored

---

## In This Guide

| Page | What You'll Learn |
| :--- | :--- |
| [Architecture](architecture) | Visual overview of systems and data flow |
| [Event Channels](event-channels) | How this sample uses Event Channels |
| [Reactive Entity Sets](reactive-entity-sets) | How this sample uses RES with Jobs and GPU |

---

## Key Files

| File | Description |
| :--- | :--- |
| `Scripts/TinyHistorySimulation.cs` | Main simulation controller |
| `Scripts/MapGenerator.cs` | Procedural map generation |
| `Scripts/MapRenderer.cs` | GPU-based province rendering |
| `Scripts/ArmyRenderer.cs` | GPU instanced army rendering |
| `Scripts/HistoryManager.cs` | Snapshot capture and restoration |
| `Scenes/TinyHistoryDemo.unity` | Main demo scene |

---

## Learn More

Want to use these patterns in your own project? Check out the feature guides:

| Guide | What You'll Learn |
| :--- | :--- |
| [Event Channels Guide]({{ '/en/guides/event-channels' | relative_url }}) | How to create and use Event Channels |
| [Variables Guide]({{ '/en/guides/variables' | relative_url }}) | How to use Reactive Variables with GPU Sync |
| [Reactive Entity Sets Guide]({{ '/en/guides/reactive-entity-sets/' | relative_url }}) | Deep dive into RES with Jobs and collections |
