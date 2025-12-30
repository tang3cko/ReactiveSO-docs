---
layout: default
title: 外部連携
nav_order: 7
has_children: true
---

# 外部連携

---

## 概要

このセクションでは、外部パッケージとの連携について説明します。これらの連携はオプションであり、対応するパッケージがインストールされている場合にのみ利用可能です。

---

## 利用可能な連携

| 連携 | 説明 |
|------|------|
| [R3](r3) | 高度なイベント処理のためのReactive Extensions対応 |

---

## 連携の仕組み

Reactive SOはUnityのアセンブリ定義の`versionDefines`を使用して、インストールされているパッケージを自動的に検出します。対応パッケージがインストールされると、追加の設定なしで連携コードが利用可能になります。

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

拡張メソッドは`Runtime/External/R3/`ディレクトリに配置され、R3がある場合のみコンパイルされます。
