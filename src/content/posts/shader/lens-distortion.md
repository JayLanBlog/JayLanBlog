---
title: "镜头畸变 (Lens Distortion)"
published: 2026-06-11
description: "Shader Showcase 后处理特效系列 — 镜头畸变 (Lens Distortion)详解，包含算法原理、参数说明与完整注释源码"
tags: ["Shader", "GLSL", "后处理", "镜头畸变", "镜头效果"]
category: "Shader分析"
draft: false
---

> 本文是 **Shader Showcase 后处理特效着色器源码详解** 系列的第 18 篇。

### 效果概述

镜头畸变着色器属于 **Distort（扭曲变形）** 类别，模拟了真实相机镜头的光学畸变效果。该着色器基于 Brown-Conrady 畸变模型的简化版本，通过径向距离的二阶和四阶多项式来控制图像的桶形畸变（Barrel Distortion，正值参数）和枕形畸变（Pincushion Distortion，负值参数）。桶形畸变使图像边缘向外膨胀，中心区域相对收缩，类似鱼眼镜头效果；枕形畸变使图像边缘向内收缩，中心区域相对膨胀。该效果常用于模拟特定镜头特征、校正镜头畸变或创造特殊的视觉风格。

### 参数说明

| 参数名 | 标签 | 类型 | 最小值 | 最大值 | 默认值 | UI 类型 | 说明 |
|--------|------|------|--------|--------|--------|---------|------|
| `uParamFloat0` | 畸变强度 | Float | -0.5 | 0.5 | 0.1 | slider | 畸变系数，正值=桶形，负值=枕形 |
| `uParamFloat1` | 缩放 | Float | 0.5 | 2.0 | 1.0 | slider | 画面缩放比例 |

### 算法原理

#### 1. Brown-Conrady 畸变模型

Brown-Conrady 模型是摄影测量学中描述镜头畸变的标准模型。其简化径向畸变公式为：

$$r_{distorted} = r \times (1 + k_1 \cdot r^2 + k_2 \cdot r^4)$$

其中 $r$ 为到图像中心的归一化距离，$k_1$ 为二阶径向畸变系数，$k_2$ 为四阶径向畸变系数。

在本着色器中，$k_2$ 被简化为 $k_1$ 的一半：

$$d(r) = 1 + k \cdot r^2 + 0.5k \cdot r^4$$

$$\mathbf{uv}_{distorted} = \frac{\mathbf{uv} \times d(r)}{zoom} \times 0.5 + 0.5$$

#### 2. 坐标空间转换

- 原始 UV 坐标范围：[0, 1]
- 以中心为原点：$\mathbf{uv} = (\text{vUV} - 0.5) \times 2$，范围 [-1, 1]
- 径向距离：$r^2 = \|\mathbf{uv}\|^2 = u_x^2 + u_y^2$
- 畸变后坐标：$\mathbf{uv}_{distorted} = \mathbf{uv} \times d(r) / zoom$
- 映射回 UV 空间：$\mathbf{uv}_{final} = \mathbf{uv}_{distorted} \times 0.5 + 0.5$

#### 3. 边界处理

畸变后的坐标通过 `clamp` 限制在 [0, 1] 范围内，超出部分采样最近边缘像素。这与 CRT 着色器的柔和暗边处理不同，镜头畸变使用简单的边缘夹紧，因为真实镜头畸变不会产生暗边。

### 逐段代码分析

#### 段落一：着色器声明与 Uniform 变量

标准声明结构。镜头畸变效果仅使用 2 个浮点参数，是所有着色器中参数最少的之一。

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

#### 段落二：坐标转换与畸变计算

将 UV 坐标以画面中心为原点归一化到 [-1, 1]，计算径向距离的平方和四次方，然后应用 Brown-Conrady 畸变公式。除以缩放参数实现画面缩放，最后映射回 [0, 1] 纹理空间。

```glsl
void main() {
    vec2 uv = (vUV - 0.5) * 2.0; // [-1, 1]

    // 径向畸变 (Brown-Conrady 模型简化版)
    float r2 = dot(uv, uv);     // 距离平方
    float r4 = r2 * r2;          // 距离四次方
    float distortion = 1.0 + uParamFloat0 * r2 + uParamFloat0 * 0.5 * r4;
    vec2 distortedUV = uv * distortion / uParamFloat1; // 应用畸变和缩放
    distortedUV = distortedUV * 0.5 + 0.5;            // 映射回[0,1]

    // 超出范围采样最近边缘
    distortedUV = clamp(distortedUV, 0.0, 1.0);

    vec3 color = texture(uInputTex, distortedUV).rgb;
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
    float uParamFloat0;  // 畸变强度
    float uParamFloat1;  // 缩放
    float uParamFloat2;  // 保留参数
    float uParamFloat3;  // 保留参数
    float uParamFloat4;  // 保留参数
    float uParamFloat5;  // 保留参数
    vec2 uResolution;    // 屏幕分辨率
    float uTime;         // 时间
    float uFrameCount;   // 帧计数
};

void main() {
    // 将UV坐标以画面中心为原点归一化到[-1, 1]范围
    vec2 uv = (vUV - 0.5) * 2.0;

    // === 径向畸变 (Brown-Conrady模型简化版) ===
    // 计算到中心的距离平方和四次方
    float r2 = dot(uv, uv);  // r^2 = x^2 + y^2
    float r4 = r2 * r2;       // r^4 = (r^2)^2
    // 畸变因子：1 + k*r^2 + 0.5k*r^4
    // k > 0: 桶形畸变（边缘向外膨胀）
    // k < 0: 枕形畸变（边缘向内收缩）
    float distortion = 1.0 + uParamFloat0 * r2 + uParamFloat0 * 0.5 * r4;
    // 应用畸变并除以缩放参数
    vec2 distortedUV = uv * distortion / uParamFloat1;
    // 将坐标从[-1,1]映射回[0,1]纹理空间
    distortedUV = distortedUV * 0.5 + 0.5;

    // 超出范围时采样最近边缘像素（clamp夹紧）
    distortedUV = clamp(distortedUV, 0.0, 1.0);

    // 使用畸变后的UV坐标采样输入纹理
    vec3 color = texture(uInputTex, distortedUV).rgb;
    outColor = vec4(color, 1.0);
}
```