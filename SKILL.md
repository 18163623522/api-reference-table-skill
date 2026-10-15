---
name: api-reference-table
description: "Generate structured API/node reference tables from engine source code, documentation pages, or data files. Produces categorized markdown tables with columns for node name, internal name, inputs (with types), outputs (with types), and practical explanations. Use when user asks to: create a reference table/速查表/对照表 for any API, extract function definitions from source code, document engine nodes/operations/functions, summarize library functions with parameters and outputs, or 'make a table like the Niagara math one'. Supports Obsidian markdown output with callout blocks."
metadata:
  version: 1.0.0
---

# API Reference Table Generator

You extract function/operation/node definitions from a source (C++ engine code, HLSL shaders, documentation pages, JSON data, Python modules) and produce **categorized reference tables** where every entry documents: what it's called, what goes in, what comes out, and what it actually does in practice.

## When to use this skill

| User says | Source type |
|---|---|
| "帮我做一个 XX 节点速查表" | Engine source / asset files |
| "总结一下这些函数的参数和用法" | Any code/API |
| "像 Niagara 数学节点那样的表格" | Engine nodes |
| "把 API 文档做成表格" | Documentation HTML/JSON |
| "提取所有 XX 操作并解释" | Source code scan |

## Output format (the contract)

Every table **must** have these columns:

```markdown
| 节点名 | 内部名 | 输入 | 输出 | 实用说明 |
|---|---|---|---|---|
| **Friendly Name** | `InternalName` | A（type）, B（type） | ReturnType | What it actually does, with formula or analogy |
```

### Column rules

1. **节点名** — The user-facing name (bold). Include symbol if it has one (e.g., **Add** `+`)
2. **内部名** — The code-level identifier in backticks (e.g., `` `Lerp` ``, `` `ArcTangent2` ``)
3. **输入** — Every parameter with type annotation. Format: `A（vec3）, B（float）`. Mark optional inputs. Mark valid ranges: `Alpha（float [0,1]）`
4. **输出** — Return type with range/constraints: `float [-1,1]`, `vec3（unit length）`, `bool`
5. **实用说明** — What it does, expressed as:
   - HLSL/C formula: `A × (1-α) + B × α`
   - Practical effect: "粒子颜色过渡的首选"
   - Warning: "⚠️ A < 0 会产生 NaN"
   - Comparison: "比 Distance 快（省 Sqrt）"

### Categorization rules

Group nodes into **functional categories** (not alphabetical):
- **基础四则运算** / **插值与范围** / **指数与对数** / **三角函数** / **向量运算** / **随机** / **几何检测** / **力与物理** / **类型转换** / **旋转** etc.
- Within each category, order by: most fundamental first, then by frequency of use
- Categories with <3 entries get merged into "其他" or a parent category

### Callout blocks (Obsidian format)

Embed `> [!tip]` or `> [!warning]` blocks for:
- **Performance tips** (e.g., "RcpFast uses GPU hardware, 12-bit precision")
- **Common pitfalls** (e.g., "Normalize on zero vector produces NaN")
- **Selection guidance** (e.g., "Slerp vs Lerp vs Nlerp — when to use which")
- **Practical recipes** (e.g., "Frac(NormalizedTime) → sawtooth wave for blinking")

## Data extraction methods

### Method 1: From engine C++ source

```
1. Find the operation definition file (grep for "FriendlyName", "BuildName", "Description")
2. Extract: display name, internal name, description text, parameter names/types
3. Parse NSLOCTEXT patterns for localized strings
```

### Method 2: From documentation page (HTML/JSON)

```
1. Fetch the page via document.json API or direct HTML
2. Parse table rows: <tr><td>Name</td><td>Default</td><td>Description</td></tr>
3. Map to output format
```

### Method 3: From engine asset files (.uasset)

```
1. List asset files in Content/DynamicInputs/ or Content/Functions/
2. File name = node name, directory = category
3. Cross-reference with source code for parameter details
```

### Method 4: From HLSL/shader source

```
1. Grep for function definitions: return_type function_name(parameters)
2. Extract signature from the declaration
3. Look for preceding comments for description
```

## Quality checklist

Before delivering, verify:
- [ ] Every entry has both display name and internal name
- [ ] Input types are annotated (not just "A, B" but "A（vec3）, B（float）")
- [ ] Output includes type AND range/constraints where applicable
- [ ] Practical explanation is useful (not just repeating the name)
- [ ] Warnings (NaN, division by zero, platform-specific) are flagged with ⚠️
- [ ] Categories are functional, not alphabetical
- [ ] Callout blocks exist for non-obvious tips
- [ ] Table is sortable by column (proper markdown table syntax)

## Obsidian integration

When writing to Obsidian vault:
- Include frontmatter with `title`, `created`, `updated`, `source`, `tags`
- Use `> [!tip]`, `> [!warning]`, `> [!note]` callouts
- Use `[[wiki-links]]` to cross-reference related notes
- Use `~~strikethrough~~` for deprecated items
- Bold the display names for visual scanning

## Example output (excerpt)

```markdown
## 向量运算

| 节点名 | 内部名 | 输入 | 输出 | 实用说明 |
|---|---|---|---|---|
| **Dot** | `Dot` | A（vec3）, B（vec3） | float | **点积**：`A·B = |A||B|cos(θ)`。=0 垂直，>0 同向，<0 反向 |
| **Cross** | `Cross` | A（vec3）, B（vec3） | vec3 | **叉积**：`A×B`。结果垂直于 A 和 B 构成的平面。用于计算法线/旋转轴 |
| **Normalize** | `Normalize` | A（vec3） | vec3（单位长度） | `A / |A|`。⚠️ 零向量输入会产生 NaN |
| **Safe Normalize** | — | A（vec3） | vec3 + bool | 安全归一化：零向量返回默认方向。**永远优先用这个** |

> [!tip] Safe vs Non-Safe
> `Normalize` 在输入为零向量时产生 NaN（0/0）。`SafeNormalize` 检测零向量并返回安全值。在 GPU Sim 中处理可能为零的速度/力向量时，**始终用 Safe 版本**。
```
