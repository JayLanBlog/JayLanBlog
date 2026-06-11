---
title: "卡通着色 (Toon Shading)"
published: 2026-06-11
description: "Shader Showcase 后处理特效系列 — 卡通着色 (Toon Shading)详解，包含算法原理、参数说明与完整注释源码"
tags: ["Shader", "GLSL", "后处理", "卡通", "非真实感渲染"]
category: "Shader分析"
draft: false
---

> 本文是 **Shader Showcase 后处理特效着色器源码详解** 系列的第 14 篇。

### 效果概述

卡通着色着色器属于 **Stylize（风格化）** 类别，实现了经典的卡通渲染（Cel Shading / Toon Shading）后处理效果。该着色器通过两个核心步骤实现卡通风格化：颜色量化（Color Posterization）将连续的色彩梯度减少为有限的色阶数，产生色块分明的卡通效果；边缘检测（Edge Detection）在量化色块的边界处绘制黑色描边轮廓线。这种效果广泛应用于动画风格化、游戏渲染和非真实感渲染（NPR）领域。

### 参数说明

| 参数名 | 标签 | 类型 | 最小值 | 最大值 | 默认值 | UI 类型 | 说明 |
|--------|------|------|--------|--------|--------|---------|------|
| `uParamFloat0` | 色阶数 | Float | 2.0 | 16.0 | 5.0 | slider | 颜色量化后的色阶数量（值越小越卡通） |
| `uParamFloat1` | 边缘阈值 | Float | 0.0 | 0.5 | 0.1 | slider | 描边的可见强度 |
| `uParamFloat2` | 描边宽度 | Float | 0.5 | 4.0 | 1.0 | slider | 边缘检测的采样距离（像素） |

### 算法原理

#### 1. 颜色量化（Posterization）

颜色量化将连续的颜色值映射到有限数量的离散色阶：

$$C_{quantized} = \left\lfloor \frac{C \times N}{N - 1} \right\rfloor \times \frac{1}{N - 1}$$

其中 $N$ 为色阶数。例如当 $N = 5$ 时，原来 [0, 1] 范围的连续值被映射为 5 个离散值：0, 0.25, 0.5, 0.75, 1.0。色阶数越少，色块效果越明显，卡通感越强。

#### 2. 边缘检测（Edge Detection）

边缘检测基于量化后颜色的邻域比较。对于当前像素，分别检查其右侧和下方邻居的量化颜色是否与当前像素不同：

$$\text{isEdge} = \begin{cases} 1 & \text{if } C_{quantized} \neq C_{right} \text{ or } C_{quantized} \neq C_{down} \\ 0 & \text{otherwise} \end{cases}$$

采样步长由描边宽度参数控制：

$$\Delta x = \frac{w}{\text{resolution}_x}, \quad \Delta y = \frac{w}{\text{resolution}_y}$$

其中 $w$ 为描边宽度参数。

#### 3. 描边渲染

检测到边缘的像素颜色被大幅压暗：

$$C_{final} = \text{mix}(C_{quantized}, C_{quantized} \times 0.15, \text{isEdge} \times \text{outline})$$

其中 `outline` 由边缘阈值参数控制（`clamp(uParamFloat1 * 4.0, 0.0, 1.0)`），0.15 的系数使边缘像素变为接近黑色的深色。

### 逐段代码分析

#### 段落一：着色器声明与 Uniform 变量

标准声明结构。卡通着色效果需要屏幕分辨率来计算边缘检测的采样步长。

```glsl
#version 460
layout(location=0) in vec2 vUV;
layout(location=0) out vec4 outColor;
layout(binding=0) uniform sampler2D uInputTex;
layout(std140, binding=1) uniform Params {
    float uParamFloat0;
    float uParamFloat1;
    float uParamFloat2;
    float uParamFloat3;
    float uParamFloat4;
    float uParamFloat5;
    vec2 uResolution;
    float uTime;
    float uFrameCount;
};
```

#### 段落二：颜色量化

对原始颜色进行量化处理，将连续颜色值映射到有限色阶。使用 `floor` 函数实现向下取整，`max(levels - 1.0, 1.0)` 防止除零错误。

```glsl
void main() {
    vec3 color = texture(uInputTex, vUV).rgb;

    // 颜色量化
    float levels = max(uParamFloat0, 2.0); // 至少2个色阶
    vec3 quantized = floor(color * levels) / max(levels - 1.0, 1.0);
    quantized = clamp(quantized, 0.0, 1.0);
```

#### 段落三：边缘检测与描边渲染

计算采样步长，分别获取右侧和下方邻居的量化颜色，通过 `notEqual` 比较检测边缘。边缘像素颜色被压暗到原来的 15%，产生黑色描边效果。

```glsl
    // 边缘检测 — 量化色块边界
    vec2 ts = 1.0 / uResolution * uParamFloat2; // 采样步长
    vec3 nr = clamp(floor(texture(uInputTex, vUV + vec2( 1, 0) * ts).rgb * levels) / max(levels - 1.0, 1.0), 0.0, 1.0);
    vec3 nd = clamp(floor(texture(uInputTex, vUV + vec2( 0, 1) * ts).rgb * levels) / max(levels - 1.0, 1.0), 0.0, 1.0);
    float isEdge = any(notEqual(quantized, nr)) || any(notEqual(quantized, nd)) ? 1.0 : 0.0;

    // 边缘变暗
    float outline = clamp(uParamFloat1 * 4.0, 0.0, 1.0);
    color = mix(quantized, quantized * 0.15, isEdge * outline);

    outColor = vec4(color, 1.0);
}
```

### 完整源码

```glsl
#version 460
// 顶点着色器传入的纹理坐标
layout(location=0) in vec2 vUV;
// 输出颜色
layout(location=0) out vec4 outColor;
// 输入纹理采样器
layout(binding=0) uniform sampler2D uInputTex;
// 统一参数块
layout(std140, binding=1) uniform Params {
    float uParamFloat0;  // 色阶数
    float uParamFloat1;  // 边缘阈值
    float uParamFloat2;  // 描边宽度
    float uParamFloat3;  // 保留参数
    float uParamFloat4;  // 保留参数
    float uParamFloat5;  // 保留参数
    vec2 uResolution;    // 屏幕分辨率
    float uTime;         // 时间
    float uFrameCount;   // 帧计数
};

void main() {
    // 采样输入纹理的原始颜色
    vec3 color = texture(uInputTex, vUV).rgb;

    // === 颜色量化（Posterization）===
    // 将连续颜色映射到有限色阶，产生色块分明的卡通效果
    float levels = max(uParamFloat0, 2.0); // 色阶数，至少为2
    vec3 quantized = floor(color * levels) / max(levels - 1.0, 1.0); // 量化
    quantized = clamp(quantized, 0.0, 1.0); // 限制范围

    // === 边缘检测 ===
    // 计算采样步长（基于描边宽度和屏幕分辨率）
    vec2 ts = 1.0 / uResolution * uParamFloat2;
    // 获取右侧邻居的量化颜色
    vec3 nr = clamp(floor(texture(uInputTex, vUV + vec2( 1, 0) * ts).rgb * levels) / max(levels - 1.0, 1.0), 0.0, 1.0);
    // 获取下方邻居的量化颜色
    vec3 nd = clamp(floor(texture(uInputTex, vUV + vec2( 0, 1) * ts).rgb * levels) / max(levels - 1.0, 1.0), 0.0, 1.0);
    // 比较当前像素与邻居的量化颜色，任一通道不同即为边缘
    float isEdge = any(notEqual(quantized, nr)) || any(notEqual(quantized, nd)) ? 1.0 : 0.0;

    // === 描边渲染 ===
    // 将边缘像素颜色压暗到15%，产生黑色描边效果
    float outline = clamp(uParamFloat1 * 4.0, 0.0, 1.0); // 描边强度
    color = mix(quantized, quantized * 0.15, isEdge * outline);

    outColor = vec4(color, 1.0);
}
```

---