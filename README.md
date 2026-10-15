# api-reference-table

A reusable skill for generating structured API/node reference tables from engine source code, documentation pages, or data files.

## What it does

Extracts function/operation/node definitions from any source and produces **categorized reference tables** where every entry documents:
- Node name (display + internal name)
- Inputs (with type annotations)
- Outputs (with range/constraints)
- Practical explanation (formula, effect, warnings)

## Example output

| 节点名 | 内部名 | 输入 | 输出 | 实用说明 |
|---|---|---|---|---|
| **Dot** | `Dot` | A（vec3）, B（vec3） | float | 点积：`A·B = |A||B|cos(θ)`。=0 垂直，>0 同向 |
| **Safe Normalize** | — | A（vec3） | vec3 + bool | 安全归一化：零向量返回默认方向。**永远优先用这个** |

## Supported sources

- Engine C++ source (grep NSLOCTEXT / FriendlyName / BuildName)
- Documentation pages (HTML table parsing)
- Engine asset files (.uasset directory structure)
- HLSL/shader source (function signature extraction)
- JSON data files

## Installation

Copy this folder to your skills directory:
```
cp -r . ~/.zcode/skills/api-reference-table/
```

## Origin

Extracted from a real project: [ue-api-reference](https://github.com/18163623522/ue-api-reference) — a full UE 5.8 API analysis that generated 430+ Niagara math node tables using this format.
