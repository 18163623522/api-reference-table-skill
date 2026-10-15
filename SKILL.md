---
name: api-reference-table
description: "Generate structured API/node reference tables from engine source code, documentation pages, or data files. Produces categorized markdown tables with columns for node name, internal name, inputs (with types), outputs (with types), and practical explanations. Use when user asks to: create a reference table/速查表/对照表 for any API, extract function definitions from source code, document engine nodes/operations/functions, summarize library functions with parameters and outputs, or 'make a table like the Niagara math one'. Supports Obsidian markdown output with callout blocks."
metadata:
  version: 1.1.0
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

---

## Table Format Variants

不同类型的节点用**不同的列结构**——不是所有表都用同一格式。以下是完整格式规范：

### 格式 A：基础运算（有公式）

```markdown
| 节点名 | 内部名 | 输入 | 输出 | 公式 |
|---|---|---|---|---|
| **Add** `+` | `Add` | A, B（任意数值类型） | 同类型 | `A + B` |
| **MultiplyAdd** | `Madd` | A, B, C | 同类型 | `(A × B) + C`（融合乘加，比分开做快） |
```

**适用**：四则运算、位运算、逻辑运算等有明确数学公式的操作。

### 格式 B：函数/效果（有实用说明）

```markdown
| 节点名 | 内部名 | 输入 | 输出 | 实用说明 |
|---|---|---|---|---|
| **Lerp** | `Lerp` | A, B（同类型）, Alpha（float [0,1]） | 同类型 | `A × (1-α) + B × α`。Alpha 0=A，1=B。支持标量/向量/颜色/位置 |
| **Clamp** | `Clamp` | A, Min, Max | 同类型 | `clamp(A, Min, Max)`。将值限制在范围内 |
```

**适用**：插值、随机、向量操作等需要解释"什么时候用"的节点。

### 格式 C：三角函数（多模式分组）

角度类函数有三种模式（周期/弧度/角度），分组展示：

```markdown
### 标准三角

| 节点名 | 内部名 | 输入 | 输出 | 公式 |
|---|---|---|---|---|
| **Sine** | `Sine` | Angle, Period | float [-1,1] | `sin(Angle × 2π/Period)` |
| **Sine (Radians)** | `Sine(Radians)` | AngleInRadians | float | `sin(A)` |
| **Sine (Degrees)** | `Sine(Degrees)` | AngleInDegrees | float | `sin(A × π/180)` |

### 反三角

| 节点名 | 内部名 | 输入 | 输出 | 说明 |
|---|---|---|---|---|
| **ArcSine (Radians)** | `ArcSine(Radians)` | A [-1,1] | float [−π/2, π/2] | `asin(A)` |
```

### 格式 D：Dynamic Input / Function 库（无内部名）

从资产目录提取的节点没有代码级内部名，省略该列：

```markdown
| 节点名 | 输入 | 输出 | 实用说明 |
|---|---|---|---|
| **Safe Normalize Vector** | Vector | vec3（单位长度） | 安全归一化：零向量返回默认方向。**永远优先用这个** |
| **Camera Fade** | Position, Near, Far | float [0,1] | 距相机 Near 内=1，Far 外=0（粒子淡出） |
```

### 格式 E：复合输出（如 Direction and Length Safe）

输出有多个值时，用行式描述：

```markdown
| 输入 | 输出 | 说明 |
|---|---|---|
| Vector（vec3） | **Direction**（vec3，单位向量） | 输入向量的方向 |
| Vector（vec3） | **Length**（float） | 输入向量的长度 |
| Vector（vec3） | **b Was Zero**（bool） | 输入是否为零向量（Safe 版本特有） |
```

### 格式 F：类型转换（简洁式）

```markdown
| 函数名 | 输入 → 输出 | 说明 |
|---|---|---|
| **Euler To Quaternion** | Euler（vec3，弧度） → Quat | 欧拉角→四元数 |
| **HSV To RGB** | H, S, V → LinearColor | HSV→RGB |
```

### 格式 G：常量/上下文（无输入）

```markdown
| 节点名 | 值/输出 | 说明 |
|---|---|---|
| **PI** | 3.14159... | 圆周率 |
| **Execution Index** | int | 当前粒子序号（0, 1, 2...） |
```

---

## 列内容规范

### 输入列写法

```
A, B（vec3）                    ← 类型在括号内
Alpha（float [0,1]）            ← 范围用方括号
A [-1,1], B（同类型）           ← 范围紧跟类型
A（≥0）                         ← 约束条件
Condition（bool）, TrueVal, FalseVal  ← 布尔参数
```

### 输出列写法

```
float                          ← 基本类型
float [-1,1]                   ← 有范围约束
float [0, Max)                 ← 半开区间
vec3（单位长度）                ← 属性说明
同类型                          ← 与输入类型一致
Position + Velocity            ← 多输出
bool + ContactPoint + Depth    ← 碰撞检测多输出
```

### 实用说明列写法

按优先级使用以下模式（可组合）：

1. **HLSL/数学公式**：`` `A × (1-α) + B × α` ``
2. **实际效果**：`粒子颜色过渡的首选`
3. **比较**：`比 Distance 快（省 Sqrt）`
4. **⚠️ 警告**：`⚠️ A < 0 会产生 NaN`
5. **用途举例**：`做喷射/散射效果`
6. **性能**：`用 GPU 硬件指令，比精确版快数倍`

---

## 分类规则

### 一级分类（## 标题）

按**功能域**分组，不是字母序：
- 基础四则 → 插值与范围 → 指数对数 → 三角 → 取整 → 向量 → 矩阵 → 随机 → 比较 → 逻辑 → 位运算

### 二级分类（### 子标题）

大类内再分：
- 三角函数 → 标准三角 / 反三角 / 角度转换
- 向量 → 基础操作 / 进阶操作 / 变换 / 复合节点
- 随机 → 均匀随机 / 高斯随机 / 种子随机 / 噪声

### 标题格式

带节点数标注：
```markdown
## 向量运算（12 个）
### 三角函数（17 个）
```

### 排序规则

在每个分类内部：**最基础的排前面，使用频率高的排前面**。

---

## Callout 块规范

在表格之间插入 Obsidian callout 提供选型指导：

### 性能对比

```markdown
> [!tip] 性能提示
> `RcpFast` 和 `RSqrt` 使用 GPU 硬件指令，在 GPU Sim 中比精确版本快数倍。精度损失在大多数视觉效果中不可察觉。
```

### 选型指南

```markdown
> [!tip] Slerp vs Lerp vs Nlerp
> - **Lerp**：直线插值，向量长度会变化。适合位置/颜色混合
> - **Slerp**：球面插值，保持单位长度。适合方向向量旋转
> - **Nlerp**：先 Lerp 再 Normalize。速度接近 Lerp，效果接近 Slerp
```

### 安全警告

```markdown
> [!warning] Normalize vs Direction and Length Safe
> `Normalize` 在零向量时产生 NaN。`Direction and Length Safe` 检测零向量并返回安全值。**始终用 Safe 版本**。
```

### 实用配方

```markdown
> [!tip] Frac 实用技巧
> `Frac(NormalizedTime)` 得到锯齿波——配合 `Abs(Frac(T) * 2 - 1)` 得到三角波，再配 Sine 得到平滑脉冲。闪烁/脉冲效果的基础。
```

---

## 数据提取方法

### 方法 1：引擎 C++ 源码

```bash
# 找操作定义文件
grep -rn "FriendlyName\|BuildName\|Description" Engine/Plugins/FX/Niagara/Source/NiagaraEditor/ --include="*.cpp" -l
# 提取完整定义
grep -n "Op->FriendlyName\|Op->Description\|Op->BuildName\|Op->CompactName" <file>
```

### 方法 2：文档页面（HTML/JSON）

```bash
# 通过 document.json API 获取
fetch('/community/api/documentation/document.json?path=<page>&lang=en-us')
# 解析 HTML 表格
re.findall(r'<tr><td><code>([^<]+)</code></td><td><code>([^<]*)</code></td><td>(.*?)</td>', html)
```

### 方法 3：引擎资产文件

```bash
# 目录名 = 分类，文件名 = 节点名
find Engine/Plugins/FX/Niagara/Content/DynamicInputs -name "*.uasset" | sed 's|.*/DynamicInputs/||'
find Engine/Plugins/FX/Niagara/Content/Functions -name "*.uasset" | sed 's|.*/Functions/||'
```

### 方法 4：HLSL/shader 源码

```bash
grep -rn "float3\|void\|bool\|struct" Shaders/Private/ --include="*.usf" -h | grep "(" | head -50
```

---

## 质量检查清单

- [ ] 每条有显示名（**加粗**）和内部名（`` `反引号` ``）
- [ ] 输入有类型标注（不只写 "A, B" 而要写 "A（vec3）, B（float）"）
- [ ] 输出有类型 AND 范围/约束
- [ ] 实用说明不只是重复名称（要有公式/效果/警告/对比）
- [ ] ⚠️ 标记了 NaN/除零/精度陷阱
- [ ] 分类是功能性的，不是字母序
- [ ] 分类标题带节点数
- [ ] Callout 块提供了选型建议（不只罗列）
- [ ] 同类节点排在一起（如 Sine 三种模式连续排列）

---

## 完整示例

```markdown
## 向量运算（12 个）

| 节点名 | 内部名 | 输入 | 输出 | 实用说明 |
|---|---|---|---|---|
| **Dot** | `Dot` | A（vec3）, B（vec3） | float | **点积**：`A·B = |A||B|cos(θ)`。=0 垂直，>0 同向，<0 反向 |
| **Cross** | `Cross` | A（vec3）, B（vec3） | vec3 | **叉积**：`A×B`。结果垂直于 A、B 构成的平面。用于法线/旋转轴 |
| **Normalize** | `Normalize` | A（vec3） | vec3（单位长度） | `A / |A|`。⚠️ 零向量产生 NaN |
| **Safe Normalize** | — | A（vec3） | vec3 | 安全归一化：零向量返回默认方向。**永远优先用这个** |
| **Length** | `Length` | A（vec3） | float | `|A| = √(x²+y²+z²)` |
| **Distance** | `DistancePos` | A, B（Position） | float | `|B - A|` 两点距离 |

> [!tip] Safe vs Non-Safe
> `Normalize` 零向量产生 NaN（0/0）。`SafeNormalize` 检测并返回安全值。GPU Sim 中**始终用 Safe 版本**。

## 高级随机

| 节点名 | 输入 | 输出 | 实用说明 |
|---|---|---|---|
| **Gaussian Random Float** | Mean, StdDev | float | **正态分布**随机。比 UniformRandom 更自然（中间多两端少）。粒子速度/大小分布首选 |
| **Random Vector In Cone** | ConeDirection, ConeAngle | vec3 | 锥形范围随机方向。做喷射/散射 |

> [!tip] Gaussian vs Uniform
> - **Uniform**：等概率——散弹枪效果
> - **Gaussian**：中间概率高——集中喷射（更自然）
> - 速度用 `Gaussian(Mean=100, StdDev=30)` 大部分粒子在 70~130 之间
```
