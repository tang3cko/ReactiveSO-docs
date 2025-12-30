---
layout: default
title: Integrations
nav_order: 7
has_children: true
---

# Integrations

---

## Purpose

This section covers integrations with external packages. These integrations are optional and only available when the corresponding package is installed.

---

## Available integrations

| Integration | Description |
|-------------|-------------|
| [R3](r3) | Reactive Extensions support for advanced event processing |

---

## How integrations work

Reactive SO uses Unity's assembly definition `versionDefines` to automatically detect installed packages. When a supported package is installed, the integration code becomes available without any manual configuration.

```json
{
    "versionDefines": [
        {
            "name": "com.cysharp.r3",
            "expression": "",
            "define": "R3_SUPPORT"
        }
    ]
}
```

The extension methods are located in `Runtime/External/R3/` directory and compiled only when R3 is present.
