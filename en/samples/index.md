---
layout: default
title: Samples
nav_order: 5.5
has_children: true
---

# Samples

This section contains documentation for the sample projects included with Reactive SO. These samples demonstrate practical applications of the architecture patterns.

You can find the source code for these samples in the package under `Samples~`. To install them into your project, use the **Package Manager** window in Unity.

1. Open **Window > Package Manager**
2. Select **Reactive SO** in the list
3. Expand the **Samples** section
4. Click **Import** next to the sample you want to explore

---

## Available Samples

| Sample | Key Concepts | Description |
| :--- | :--- | :--- |
| [Basic Demo (Counter)](basic-demo) | Event Channels, Variables | A simple counter UI demonstrating decoupled communication between buttons, logic, and display. |
| [GPU Sync (Motes)](mote-demo) | GPU Sync, Variables, Compute Shaders | Demonstrates how to sync VariableSO values directly to shaders for high-performance visual effects. |
| [Runtime Sets Demo](runtime-sets-demo) | Runtime Sets, Dynamic Lists | Shows how to manage dynamic collections of objects (like spawned enemies) without manager singletons. |
| [Tiny History Demo](tiny-history-demo/) | Event Channels, Reactive Entity Sets, GPU Sync | An integrated example: nation simulation with timeline navigation, demonstrating Jobs integration and history snapshots. |
