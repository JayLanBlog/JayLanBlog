---
title: "边缘检测 (Edge Detection)"
published: 2026-06-11
description: "Shader Showcase 后处理特效系列 — 边缘检测 (Edge Detection)详解，包含算法原理、参数说明与完整注释源码"
tags: ["Shader", "GLSL", "后处理", "边缘检测", "图像处理"]
category: "Shader分析"
draft: false
---

> 本文是 **Shader Showcase 后处理特效着色器源码详解** 系列的第 5 篇。

### 效果概述

边缘检测是图像处理中的经典操作，属于 **风格化** 类别。本实现采用 **Sobel 算子**，通过计算图像在水平和垂直方向上的亮度梯度来检测边缘。检测到的边缘可以以灰度或彩色两种模式显示。该效果常用于非真实感渲染（NPR）、卡通渲染的轮廓线提取等场景。

### 参数说明

| 参数名 | UI 标签 | 类型 | 最小值 | 最大值 | 默认值 | UI 控件 |
|--------|---------|------|--------|--------|--------|---------|
| `uParamFloat0` | 边缘强度 | Float | 0.0 | 5.0 | 1.5 | 滑块 |
| `uParamFloat1` | 显示颜色 | Float | 0.0 | 1.0 | 0.0 | 滑块 |

- **边缘强度**：控制边缘检测的灵敏度/亮度倍率
- **显示颜色**：0.0 时输出灰度边缘，1.0 时输出彩色边缘（用原色乘以边缘值）

### 算法原理

**Sobel 算子**使用两个 3x3 卷积核分别检测水平和垂直方向的梯度：

**水平方向核 $G_x$：**

$$
G_x = \begin{bmatrix} -1 & 0 & 1 \\ -2 & 0 & 2 \\ -1 & 0 & 1 \end{bmatrix}
$$

**垂直方向核 $G_y$：**

$$
G_y = \begin{bmatrix} -1 & -2 & -1 \\ 0 & 0 & 0 \\ 1 & 2 & 1 \end{bmatrix}
$$

对 3x3 邻域的 8 个像素（除中心像素外）分别计算亮度值，然后按 Sobel 核权重求和：

$$
S_x = -1 \cdot tl + 1 \cdot tr - 2 \cdot l + 2 \cdot r - 1 \cdot bl + 1 \cdot br
$$

$$
S_y = -1 \cdot tl - 2 \cdot t - 1 \cdot tr + 1 \cdot bl + 2 \cdot b + 1 \cdot br
$$

最终梯度幅值为：

$$
\text{edge} = \sqrt{S_x^2 + S_y^2} \times \text{EdgeStrength}
$$

### 逐段代码分析

**参数读取与提前退出：**

```glsl
void main() {
    vec3 color = texture(uInputTex, vUV).rgb;

    float EdgeStrength = uParamFloat0;
    float ShowColor = uParamFloat1;

    // 提前退出：边缘强度为 0 时直接输出原图
    if (EdgeStrength <= 0.0) {
        outColor = vec4(color, 1.0);
        return;
    }
```

**3x3 邻域采样与亮度计算：**

对当前像素周围的 8 个邻域像素进行采样，使用 BT.709 亮度系数将 RGB 转换为灰度值。

```glsl
    vec2 texelSize = 1.0 / uResolution;
    vec3 lumaCoeff = vec3(0.2126, 0.7152, 0.0722);

    // 采样 3x3 邻域并计算亮度
    float tl = dot(texture(uInputTex, vUV + vec2(-1.0, -1.0) * texelSize).rgb, lumaCoeff);
    float t  = dot(texture(uInputTex, vUV + vec2( 0.0, -1.0) * texelSize).rgb, lumaCoeff);
    float tr = dot(texture(uInputTex, vUV + vec2( 1.0, -1.0) * texelSize).rgb, lumaCoeff);
    float l  = dot(texture(uInputTex, vUV + vec2(-1.0,  0.0) * texelSize).rgb, lumaCoeff);
    float r  = dot(texture(uInputTex, vUV + vec2( 1.0,  0.0) * texelSize).rgb, lumaCoeff);
    float bl = dot(texture(uInputTex, vUV + vec2(-1.0,  1.0) * texelSize).rgb, lumaCoeff);
    float b  = dot(texture(uInputTex, vUV + vec2( 0.0,  1.0) * texelSize).rgb, lumaCoeff);
    float br = dot(texture(uInputTex, vUV + vec2( 1.0,  1.0) * texelSize).rgb, lumaCoeff);
```

**Sobel 梯度计算与输出：**

```glsl
    // Sobel 水平和垂直梯度
    float sx = tl * -1.0 + tr * 1.0 + l * -2.0 + r * 2.0 + bl * -1.0 + br * 1.0;
    float sy = tl * -1.0 + t * -2.0 + tr * -1.0 + bl * 1.0 + b * 2.0 + br * 1.0;

    // 梯度幅值 = sqrt(sx^2 + sy^2)
    float edge = sqrt(sx * sx + sy * sy) * EdgeStrength;

    // 在灰度边缘和彩色边缘之间混合
    vec3 edgeColor = mix(vec3(edge), color * edge, ShowColor);

    outColor = vec4(edgeColor, 1.0);
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
    float uParamFloat0;   // 边缘强度
    float uParamFloat1;   // 显示颜色模式
    float uParamFloat2;   // 未使用
    float uParamFloat3;   // 未使用
    float uParamFloat4;   // 未使用
    float uParamFloat5;   // 未使用
    vec2  uResolution;    // 屏幕分辨率
    float uTime;          // 运行时间
    float uFrameCount;    // 帧计数
};

void main() {
    // 采样当前像素的原始颜色
    vec3 color = texture(uInputTex, vUV).rgb;

    // 读取边缘检测参数
    float EdgeStrength = uParamFloat0;  // 边缘强度
    float ShowColor = uParamFloat1;       // 颜色显示模式

    // 提前退出：边缘强度为 0 时直接输出原图
    if (EdgeStrength <= 0.0) {
        outColor = vec4(color, 1.0);
        return;
    }

    // 计算纹素大小
    vec2 texelSize = 1.0 / uResolution;
    // BT.709 亮度系数，用于将 RGB 转为灰度
    vec3 lumaCoeff = vec3(0.2126, 0.7152, 0.0722);

    // 采样 3x3 邻域的 8 个像素并计算亮度值
    // tl = 左上, t = 上, tr = 右上
    float tl = dot(texture(uInputTex, vUV + vec2(-1.0, -1.0) * texelSize).rgb, lumaCoeff);
    float t  = dot(texture(uInputTex, vUV + vec2( 0.0, -1.0) * texelSize).rgb, lumaCoeff);
    float tr = dot(texture(uInputTex, vUV + vec2( 1.0, -1.0) * texelSize).rgb, lumaCoeff);
    // l = 左, r = 右
    float l  = dot(texture(uInputTex, vUV + vec2(-1.0,  0.0) * texelSize).rgb, lumaCoeff);
    float r  = dot(texture(uInputTex, vUV + vec2( 1.0,  0.0) * texelSize).rgb, lumaCoeff);
    // bl = 左下, b = 下, br = 右下
    float bl = dot(texture(uInputTex, vUV + vec2(-1.0,  1.0) * texelSize).rgb, lumaCoeff);
    float b  = dot(texture(uInputTex, vUV + vec2( 0.0,  1.0) * texelSize).rgb, lumaCoeff);
    float br = dot(texture(uInputTex, vUV + vec2( 1.0,  1.0) * texelSize).rgb, lumaCoeff);

    // Sobel 水平梯度 Gx
    // 核: [-1 0 1; -2 0 2; -1 0 1]
    float sx = tl * -1.0 + tr * 1.0 + l * -2.0 + r * 2.0 + bl * -1.0 + br * 1.0;
    // Sobel 垂直梯度 Gy
    // 核: [-1 -2 -1; 0 0 0; 1 2 1]
    float sy = tl * -1.0 + t * -2.0 + tr * -1.0 + bl * 1.0 + b * 2.0 + br * 1.0;

    // 梯度幅值 = sqrt(Gx^2 + Gy^2)，乘以强度
    float edge = sqrt(sx * sx + sy * sy) * EdgeStrength;

    // 混合模式：ShowColor=0 时为灰度边缘，ShowColor=1 时为彩色边缘
    vec3 edgeColor = mix(vec3(edge), color * edge, ShowColor);

    outColor = vec4(edgeColor, 1.0);
}
```

---