---
title: "万花筒效果 (Kaleidoscope)"
published: 2026-06-11
description: "Shader Showcase 后处理特效系列 — 万花筒效果 (Kaleidoscope)详解，包含算法原理、参数说明与完整注释源码"
tags: ["Shader", "GLSL", "后处理", "万花筒", "几何"]
category: "Shader分析"
draft: false
---

> 本文是 **Shader Showcase 后处理特效着色器源码详解** 系列的第 12 篇。

### 效果概述

万花筒着色器属于 **Distort（扭曲变形）** 类别，模拟了光学万花筒的径向对称效果。该着色器将画面从中心向外划分为若干等角扇形区域，每个扇形内的图像通过镜像翻转与其他扇形共享，从而产生类似万花筒的对称图案。同时支持自动旋转和缩放功能，可创建动态变化的几何对称效果。这种效果常用于音乐可视化、艺术风格化视频和创意后期处理中。

### 参数说明

| 参数名 | 标签 | 类型 | 最小值 | 最大值 | 默认值 | UI 类型 | 说明 |
|--------|------|------|--------|--------|--------|---------|------|
| `uParamFloat0` | 对称段数 | Float | 2.0 | 12.0 | 6.0 | slider | 万花筒的对称扇形数量 |
| `uParamFloat1` | 旋转速度 | Float | 0.0 | 2.0 | 0.3 | slider | 自动旋转的角速度 |
| `uParamFloat2` | 缩放 | Float | 0.5 | 3.0 | 1.0 | slider | 画面缩放比例 |

### 算法原理

#### 1. 极坐标转换

首先将屏幕空间的纹理坐标从笛卡尔坐标转换为极坐标：

$$r = \|\mathbf{uv}\| = \sqrt{x^2 + y^2}$$

$$\theta = \text{atan2}(y, x)$$

其中 $\mathbf{uv}$ 是以画面中心为原点的归一化坐标。

#### 2. 角度折叠与镜像

将极角 $\theta$ 折叠到一个扇形区域内，实现对称效果：

$$\theta_{seg} = \frac{2\pi}{N}$$

$$\theta' = \text{mod}(\theta + \omega \cdot t, \theta_{seg})$$

$$\theta'' = \begin{cases} \theta' & \text{if } \theta' \leq \theta_{seg}/2 \\ \theta_{seg} - \theta' & \text{if } \theta' > \theta_{seg}/2 \end{cases}$$

其中 $N$ 为对称段数，$\omega$ 为旋转速度，$t$ 为时间。`mod` 操作将任意角度映射到单个扇形内，条件判断实现扇形内的镜像翻转，使得相邻扇形互为镜像。

#### 3. 坐标重建

将折叠后的极坐标转回笛卡尔坐标：

$$\mathbf{uv}_{new} = (\cos\theta'' \cdot r \cdot 0.5 + 0.5, \sin\theta'' \cdot r \cdot 0.5 + 0.5)$$

乘以 0.5 并偏移 0.5 是将 [-1, 1] 范围映射回 [0, 1] 的纹理坐标空间。

### 逐段代码分析

#### 段落一：着色器声明与 Uniform 变量

标准声明结构，定义了输入输出和参数块。万花筒效果不需要帧计数参数，但保留了统一的参数块结构。

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

#### 段落二：极坐标转换与角度折叠

将纹理坐标以画面中心为原点进行归一化（除以缩放参数），然后转换为极坐标。通过 `mod` 运算将角度折叠到单个扇形内，再通过条件判断实现镜像翻转，使得每个扇形内的图案互为镜像。

```glsl
#define PI 3.14159265359

void main() {
    vec2 uv = (vUV - 0.5) * 2.0 / uParamFloat2; // 以中心为原点归一化
    float angle = atan(uv.y, uv.x);                // 极角
    float radius = length(uv);                      // 极径

    // 将角度映射到 [0, 2PI/segments]
    float segAngle = PI * 2.0 / uParamFloat0;      // 单个扇形的角度范围
    angle = mod(angle + uTime * uParamFloat1, segAngle); // 折叠+旋转
    if (angle > segAngle * 0.5) angle = segAngle - angle;  // 镜像翻转
```

#### 段落三：坐标重建与纹理采样

将折叠后的极坐标转回笛卡尔坐标，映射回 [0, 1] 纹理空间，使用 `clamp` 防止越界采样，最后从输入纹理采样输出颜色。

```glsl
    // 重建UV (镜像)
    vec2 newUV = vec2(cos(angle), sin(angle)) * radius * 0.5 + 0.5;
    newUV = clamp(newUV, 0.0, 1.0);

    vec3 color = texture(uInputTex, newUV).rgb;
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
    float uParamFloat0;  // 对称段数
    float uParamFloat1;  // 旋转速度
    float uParamFloat2;  // 缩放
    float uParamFloat3;  // 保留参数
    float uParamFloat4;  // 保留参数
    float uParamFloat5;  // 保留参数
    vec2 uResolution;    // 屏幕分辨率
    float uTime;         // 时间
    float uFrameCount;   // 帧计数
};

#define PI 3.14159265359 // 圆周率常量

void main() {
    // 将纹理坐标以画面中心为原点进行归一化，并应用缩放
    vec2 uv = (vUV - 0.5) * 2.0 / uParamFloat2;
    // 计算极坐标：角度和半径
    float angle = atan(uv.y, uv.x); // 极角（弧度）
    float radius = length(uv);       // 极径

    // 计算单个扇形的角度范围
    float segAngle = PI * 2.0 / uParamFloat0;
    // 将角度折叠到单个扇形内，并加上时间偏移实现旋转
    angle = mod(angle + uTime * uParamFloat1, segAngle);
    // 镜像翻转：超过扇形一半时取反，产生对称效果
    if (angle > segAngle * 0.5) angle = segAngle - angle;

    // 将折叠后的极坐标重建为笛卡尔坐标
    // 乘以0.5并偏移0.5，将[-1,1]映射回[0,1]纹理空间
    vec2 newUV = vec2(cos(angle), sin(angle)) * radius * 0.5 + 0.5;
    // 限制UV范围，防止越界采样
    newUV = clamp(newUV, 0.0, 1.0);

    // 使用重建的UV坐标采样输入纹理
    vec3 color = texture(uInputTex, newUV).rgb;
    outColor = vec4(color, 1.0);
}
```

---