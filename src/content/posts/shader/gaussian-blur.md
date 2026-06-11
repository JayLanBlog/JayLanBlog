---
title: "高斯模糊 (Gaussian Blur)"
published: 2026-06-11
description: "Shader Showcase 后处理特效系列 — 高斯模糊 (Gaussian Blur)详解，包含算法原理、参数说明与完整注释源码"
tags: ["Shader", "GLSL", "后处理", "模糊", "图像处理"]
category: "Shader分析"
draft: false
---

> 本文是 **Shader Showcase 后处理特效着色器源码详解** 系列的第 3 篇。

### 效果概述

高斯模糊是最经典的图像模糊算法，属于 **模糊** 类别。它通过对当前像素周围的邻域进行加权平均来实现平滑效果，权重由二维高斯函数决定。距离中心越远的像素权重越低，从而在模糊的同时保留一定的图像结构。本实现采用固定 9x9 采样核，通过参数控制模糊半径和混合强度。

### 参数说明

| 参数名 | UI 标签 | 类型 | 最小值 | 最大值 | 默认值 | UI 控件 |
|--------|---------|------|--------|--------|--------|---------|
| `uParamFloat0` | 模糊半径 | Float | 1.0 | 20.0 | 5.0 | 滑块 |
| `uParamFloat1` | 模糊强度 | Float | 0.0 | 1.0 | 1.0 | 滑块 |

- **模糊半径**：控制高斯核的 sigma 值和采样偏移量，值越大模糊越强
- **模糊强度**：控制原图与模糊图之间的混合比例（0 = 原图，1 = 完全模糊）

### 算法原理

**二维高斯卷积：**

高斯核的权重函数为：

$$
w(x, y) = e^{-\frac{x^2 + y^2}{2\sigma^2}}
$$

其中 $\sigma = \text{BlurRadius} \times 0.5$。

采样偏移量为：

$$
\Delta_{uv} = (x, y) \times \text{texelSize} \times \text{BlurRadius} \times 0.3
$$

最终通过 `mix()` 函数在原图和模糊结果之间线性插值：

$$
\text{output} = \text{mix}(\text{original}, \text{blurred}, \text{BlurStrength})
$$

### 逐段代码分析

**参数读取与提前退出：**

```glsl
void main() {
    vec3 color = texture(uInputTex, vUV).rgb;

    float BlurRadius = max(uParamFloat0, 0.5); // 防止除零
    float BlurStrength = uParamFloat1;

    // 提前退出：无模糊时直接输出原图
    if (BlurStrength <= 0.0) {
        outColor = vec4(color, 1.0);
        return;
    }
```

**高斯模糊计算：**

使用固定 9x9 核（范围 -4 到 4），sigma 和采样偏移均由 BlurRadius 控制。

```glsl
    // 高斯模糊：半径控制核大小和采样扩散
    vec2 texelSize = 1.0 / uResolution;
    vec3 result = vec3(0.0);
    float total = 0.0;
    
    // sigma = 半径 * 0.5
    float sigma = BlurRadius * 0.5;
    float sigma2 = 2.0 * sigma * sigma;

    // 固定 9x9 采样核
    for (int x = -4; x <= 4; x++) {
        for (int y = -4; y <= 4; y++) {
            float dist2 = float(x * x + y * y);
            float w = exp(-dist2 / sigma2);
            // 偏移量 = 整数偏移 * 纹素大小 * 半径 * 0.3
            vec2 offset = vec2(float(x), float(y)) * texelSize * BlurRadius * 0.3;
            result += texture(uInputTex, vUV + offset).rgb * w;
            total += w;
        }
    }
    result /= max(total, 0.001);
```

**混合输出：**

```glsl
    // 强度控制原图与模糊结果的混合
    outColor = vec4(mix(color, result, BlurStrength), 1.0);
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
    float uParamFloat0;   // 模糊半径
    float uParamFloat1;   // 模糊强度
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

    // 读取模糊参数，半径最小值 0.5 防止除零
    float BlurRadius = max(uParamFloat0, 0.5);
    float BlurStrength = uParamFloat1;

    // 提前退出：模糊强度为 0 时直接输出原图
    if (BlurStrength <= 0.0) {
        outColor = vec4(color, 1.0);
        return;
    }

    // 计算纹素大小
    vec2 texelSize = 1.0 / uResolution;
    vec3 result = vec3(0.0);
    float total = 0.0;
    
    // sigma 与模糊半径成正比
    float sigma = BlurRadius * 0.5;
    float sigma2 = 2.0 * sigma * sigma;

    // 固定 9x9 高斯核卷积
    for (int x = -4; x <= 4; x++) {
        for (int y = -4; y <= 4; y++) {
            // 采样点到中心的距离平方
            float dist2 = float(x * x + y * y);
            // 高斯权重
            float w = exp(-dist2 / sigma2);
            // 采样偏移量，乘以半径和 0.3 系数控制扩散程度
            vec2 offset = vec2(float(x), float(y)) * texelSize * BlurRadius * 0.3;
            result += texture(uInputTex, vUV + offset).rgb * w;
            total += w;
        }
    }
    // 权重归一化
    result /= max(total, 0.001);

    // 在原图和模糊结果之间按强度混合
    outColor = vec4(mix(color, result, BlurStrength), 1.0);
}
```

---