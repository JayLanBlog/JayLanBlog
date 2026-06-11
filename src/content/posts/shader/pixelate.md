---
title: "像素化效果 (Pixelate)"
published: 2026-06-11
description: "Shader Showcase 后处理特效系列 — 像素化效果 (Pixelate)详解，包含算法原理、参数说明与完整注释源码"
tags: ["Shader", "GLSL", "后处理", "像素化", "图像处理"]
category: "Shader分析"
draft: false
---

> 本文是 **Shader Showcase 后处理特效着色器源码详解** 系列的第 7 篇。

### 效果概述

像素化（马赛克）效果将图像分割为固定大小的方块，每个方块内的所有像素显示为同一颜色，属于 **风格化** 类别。这种效果模拟了低分辨率显示器的像素网格外观，常用于复古游戏风格、隐私遮挡、以及艺术化处理等场景。本实现通过将 UV 坐标量化到方块网格上来实现。

### 参数说明

| 参数名 | UI 标签 | 类型 | 最小值 | 最大值 | 默认值 | UI 控件 |
|--------|---------|------|--------|--------|--------|---------|
| `uParamFloat0` | 像素大小 | Float | 1.0 | 50.0 | 8.0 | 滑块 |

- **像素大小**：每个马赛克方块的边长（以纹素为单位），值越大像素化越明显

### 算法原理

像素化的核心是 **UV 坐标量化**：

1. 计算像素化后的网格数量：

$$
\text{pixelCount} = \frac{\text{resolution}}{\max(\text{blockSize}, 1)}
$$

2. 将连续的 UV 坐标映射到离散的网格单元，取每个单元的中心点：

$$
\text{uv}_{\text{quantized}} = \frac{\lfloor \text{uv} \times \text{pixelCount} \rfloor + 0.5}{\text{pixelCount}}
$$

`floor()` 将 UV 坐标对齐到网格的左下角，`+ 0.5` 偏移到方块中心。使用方块中心采样而非角落，可以利用 GPU 的双线性滤波避免方块边缘的颜色渗透。

### 逐段代码分析

**UV 量化与方块中心采样：**

```glsl
void main() {
    // 计算像素化后的网格数量（每个维度有多少个方块）
    vec2 pixelCount = uResolution / max(uParamFloat0, 1.0);
    // 将 UV 坐标量化到方块中心
    // floor(uv * pixelCount) 对齐到网格左下角
    // + 0.5 偏移到方块中心（配合线性滤波避免边缘渗透）
    vec2 uv = (floor(vUV * pixelCount) + 0.5) / pixelCount;
    // 使用量化后的 UV 采样纹理
    vec3 color = texture(uInputTex, uv).rgb;
    outColor = vec4(color, 1.0);
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
    float uParamFloat0;   // 像素大小
    float uParamFloat1;   // 未使用
    float uParamFloat2;   // 未使用
    float uParamFloat3;   // 未使用
    float uParamFloat4;   // 未使用
    float uParamFloat5;   // 未使用
    vec2  uResolution;    // 屏幕分辨率
    float uTime;          // 运行时间
    float uFrameCount;    // 帧计数
};

void main() {
    // 计算像素化后的网格数量
    // 分辨率除以方块大小，得到每个维度的方块数
    vec2 pixelCount = uResolution / max(uParamFloat0, 1.0);
    // 将连续 UV 坐标量化到方块中心
    // 步骤1: vUV * pixelCount -> 将 UV 映射到网格坐标
    // 步骤2: floor() -> 向下取整对齐到网格左下角
    // 步骤3: + 0.5 -> 偏移到方块中心（利用线性滤波避免边缘颜色渗透）
    // 步骤4: / pixelCount -> 映射回 [0,1] 的 UV 空间
    vec2 uv = (floor(vUV * pixelCount) + 0.5) / pixelCount;
    // 使用量化后的 UV 采样纹理，同一方块内的所有像素获得相同颜色
    vec3 color = texture(uInputTex, uv).rgb;
    outColor = vec4(color, 1.0);
}
```

---