---
title: "浮雕效果 (Emboss)"
published: 2026-06-11
description: "Shader Showcase 后处理特效系列 — 浮雕效果 (Emboss)详解，包含算法原理、参数说明与完整注释源码"
tags: ["Shader", "GLSL", "后处理", "浮雕", "图像处理"]
category: "Shader分析"
draft: false
---

> 本文是 **Shader Showcase 后处理特效着色器源码详解** 系列的第 6 篇。

### 效果概述

浮雕效果模拟光线从特定方向照射时产生的凹凸纹理感，属于 **风格化** 类别。它通过计算当前像素与沿指定方向偏移的邻居像素之间的差值，并将结果偏移到中间灰度（0.5），从而产生类似浮雕的立体视觉效果。方向角度可调，使浮雕的光照方向可以任意旋转。

### 参数说明

| 参数名 | UI 标签 | 类型 | 最小值 | 最大值 | 默认值 | UI 控件 |
|--------|---------|------|--------|--------|--------|---------|
| `uParamFloat0` | 强度 | Float | 0.0 | 10.0 | 4.0 | 滑块 |
| `uParamFloat1` | 角度 | Float | 0.0 | 360.0 | 135.0 | 滑块 |

- **强度**：控制浮雕效果的深浅程度，值越大凹凸感越强
- **角度**：控制浮雕光照方向（0-360 度），默认 135 度为左上方光源

### 算法原理

浮雕的核心是 **方向性差分**：

1. 根据角度参数计算方向向量：

$$
\vec{d} = (\cos(\theta), \sin(\theta)) \times \text{texelSize}
$$

2. 采样当前像素和沿方向偏移一个纹素的邻居像素，计算差值：

$$
\text{diff} = (\text{neighbor} - \text{center}) \times \text{Strength} + 0.5
$$

加 0.5 的目的是将差值从 `[-0.5, 0.5]` 偏移到 `[0, 1]`，使无变化的区域呈现中灰色（0.5），而边缘区域呈现亮色或暗色，形成浮雕效果。

### 逐段代码分析

**方向向量计算：**

将角度转换为弧度，计算方向单位向量并乘以纹素大小。

```glsl
void main() {
    vec2 texelSize = 1.0 / uResolution;
    // 将角度参数从度数转换为弧度
    float rad = radians(uParamFloat1);
    // 计算方向向量并缩放到纹素大小
    vec2 dir = vec2(cos(rad), sin(rad)) * texelSize;
```

**浮雕差分计算：**

采样当前像素和偏移邻居，计算差值并偏移到中间灰度。

```glsl
    vec3 color = texture(uInputTex, vUV).rgb;
    vec3 neighbor = texture(uInputTex, vUV + dir).rgb;
    // 差值乘以强度，加 0.5 偏移到中间灰度
    vec3 diff = (neighbor - color) * uParamFloat0 + 0.5;
    outColor = vec4(clamp(diff, 0.0, 1.0), 1.0);
}
```

### 完整源码

```glsl
#version 460
// 输入：插值纹理坐标
layout(location=0) in vec2 vUV;
// 输出：最终像素颜色
layout(location=0) out vec4 outColor;

// 输入纹理
layout(binding=0) uniform sampler2D uInputTex;

// 统一参数块
layout(std140, binding=1) uniform Params {
    float uParamFloat0;   // 浮雕强度
    float uParamFloat1;   // 光照角度
    float uParamFloat2;   // 未使用
    float uParamFloat3;   // 未使用
    float uParamFloat4;   // 未使用
    float uParamFloat5;   // 未使用
    vec2  uResolution;    // 屏幕分辨率
    float uTime;          // 运行时间
    float uFrameCount;    // 帧计数
};

void main() {
    // 计算单个纹素的大小
    vec2 texelSize = 1.0 / uResolution;
    // 将角度从度数转换为弧度
    float rad = radians(uParamFloat1);
    // 计算浮雕方向向量（单位圆上的点），缩放到纹素大小
    vec2 dir = vec2(cos(rad), sin(rad)) * texelSize;

    // 采样当前像素颜色
    vec3 color = texture(uInputTex, vUV).rgb;
    // 沿方向偏移一个纹素采样邻居颜色
    vec3 neighbor = texture(uInputTex, vUV + dir).rgb;
    // 计算差值：邻居 - 中心，乘以强度，加 0.5 偏移到中间灰度
    // 正差值 -> 亮色（凸起），负差值 -> 暗色（凹陷）
    vec3 diff = (neighbor - color) * uParamFloat0 + 0.5;
    // clamp 到 [0,1] 防止溢出
    outColor = vec4(clamp(diff, 0.0, 1.0), 1.0);
}
```

---