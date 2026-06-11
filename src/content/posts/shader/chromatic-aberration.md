---
title: "色差效果 (Chromatic Aberration)"
published: 2026-06-11
description: "Shader Showcase 后处理特效系列 — 色差效果 (Chromatic Aberration)详解，包含算法原理、参数说明与完整注释源码"
tags: ["Shader", "GLSL", "后处理", "色差", "镜头效果"]
category: "Shader分析"
draft: false
---

> 本文是 **Shader Showcase 后处理特效着色器源码详解** 系列的第 9 篇。

### 效果概述

色差（Chromatic Aberration）模拟真实光学镜头中因色散导致的 RGB 通道错位现象，属于 **扭曲** 类别。由于不同波长的光在透镜中折射率不同，红、绿、蓝三个通道在图像边缘会产生不同程度的偏移，形成彩色边缘。本实现支持径向和线性两种色差模式，径向模式下偏移量随距画面中心的距离增大而增大，更接近真实镜头效果。

### 参数说明

| 参数名 | UI 标签 | 类型 | 最小值 | 最大值 | 默认值 | UI 控件 |
|--------|---------|------|--------|--------|--------|---------|
| `uParamFloat0` | 色差强度 | Float | 0.0 | 20.0 | 3.0 | 滑块 |
| `uParamFloat1` | 径向程度 | Float | 0.0 | 1.0 | 1.0 | 滑块 |

- **色差强度**：控制 RGB 通道偏移的总体大小
- **径向程度**：0.0 时为纯线性色差（均匀偏移），1.0 时为纯径向色差（边缘偏移大，中心无偏移）

### 算法原理

**色差的核心是 RGB 通道分离采样：**

1. 计算当前像素到画面中心的方向和距离：

$$
\vec{c} = \text{vUV} - 0.5, \quad d = |\vec{c}| \times 2, \quad \hat{d} = \text{normalize}(\vec{c} + \epsilon)
$$

2. 计算总偏移量，混合径向和线性分量：

$$
\text{offset}_{\text{base}} = \text{Intensity} \times \text{texelSize}_x
$$

$$
\text{offset}_{\text{radial}} = \text{offset}_{\text{base}} \times d \times \text{RadialFactor}
$$

$$
\text{offset}_{\text{linear}} = \text{offset}_{\text{base}} \times (1 - \text{RadialFactor})
$$

$$
\text{offset}_{\text{total}} = \text{offset}_{\text{radial}} + \text{offset}_{\text{linear}}
$$

3. 对三个通道分别在不同位置采样：

$$
R = \text{sample}(\text{vUV} + \hat{d} \times \text{offset}_{\text{total}})  \\
G = \text{sample}(\text{vUV})  \\
B = \text{sample}(\text{vUV} - \hat{d} \times \text{offset}_{\text{total}})
$$

红色通道沿径向向外偏移，蓝色通道向内偏移，绿色通道保持不变。这种不对称偏移模拟了光学色散中长波长（红）和短波长（蓝）折射率差异。

### 逐段代码分析

**方向和距离计算：**

```glsl
void main() {
    vec2 texelSize = 1.0 / uResolution;
    // 计算到画面中心的向量
    vec2 center = vUV - 0.5;
    // 归一化距离 [0, 1]（对角线方向最大为 ~0.707，乘 2 扩展范围）
    float dist = length(center) * 2.0;
    // 计算径向方向单位向量，加微小值防止中心点除零
    vec2 dir = normalize(center + 0.0001);
```

**偏移量混合计算：**

```glsl
    // 基础偏移量 = 强度 * 水平纹素大小
    float offset = uParamFloat0 * texelSize.x;
    // 径向偏移：与到中心的距离成正比
    float radialOffset = offset * dist * uParamFloat1;
    // 线性偏移：与距离无关的均匀偏移
    float linearOffset = offset * (1.0 - uParamFloat1);
    // 总偏移 = 径向 + 线性
    float totalOffset = radialOffset + linearOffset;
```

**RGB 通道分离采样：**

```glsl
    // 红色通道沿径向向外偏移（模拟长波长折射率小）
    float r = texture(uInputTex, vUV + dir * totalOffset).r;
    // 绿色通道保持原位
    float g = texture(uInputTex, vUV).g;
    // 蓝色通道沿径向向内偏移（模拟短波长折射率大）
    float b = texture(uInputTex, vUV - dir * totalOffset).b;

    outColor = vec4(r, g, b, 1.0);
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
    float uParamFloat0;   // 色差强度
    float uParamFloat1;   // 径向程度
    float uParamFloat2;   // 未使用
    float uParamFloat3;   // 未使用
    float uParamFloat4;   // 未使用
    float uParamFloat5;   // 未使用
    vec2  uResolution;    // 屏幕分辨率
    float uTime;          // 运行时间
    float uFrameCount;    // 帧计数
};

void main() {
    // 计算纹素大小
    vec2 texelSize = 1.0 / uResolution;
    // 计算当前像素到画面中心的向量
    vec2 center = vUV - 0.5;
    // 归一化距离，乘以 2 扩展到 [0, ~1.414] 范围
    float dist = length(center) * 2.0;
    // 计算径向方向单位向量，加 0.0001 防止中心点 normalize 除零
    vec2 dir = normalize(center + 0.0001);

    // 基础偏移量 = 色差强度 * 水平纹素大小
    float offset = uParamFloat0 * texelSize.x;
    // 径向偏移分量：偏移量与到中心距离成正比
    // 距离越大（越靠近边缘），偏移越大
    float radialOffset = offset * dist * uParamFloat1;
    // 线性偏移分量：均匀偏移，与距离无关
    float linearOffset = offset * (1.0 - uParamFloat1);
    // 总偏移量 = 径向偏移 + 线性偏移
    float totalOffset = radialOffset + linearOffset;

    // RGB 通道分离采样
    // 红色通道：沿径向向外偏移（模拟长波长光折射率较小）
    float r = texture(uInputTex, vUV + dir * totalOffset).r;
    // 绿色通道：保持原位不偏移
    float g = texture(uInputTex, vUV).g;
    // 蓝色通道：沿径向向内偏移（模拟短波长光折射率较大）
    float b = texture(uInputTex, vUV - dir * totalOffset).b;

    // 组合三个通道输出
    outColor = vec4(r, g, b, 1.0);
}
```

---

> **文档说明：** 本文档基于 Shader Showcase 项目中的 9 个后处理特效着色器编写，所有着色器使用 GLSL 460 版本，共享统一的 UBO 参数块布局和全屏四边形顶点着色器。每个特效均为单通道（single-pass）实现，通过 `effect.json` 配置文件定义参数范围和 UI 控件。