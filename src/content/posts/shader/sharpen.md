---
title: "锐化效果 (Sharpen)"
published: 2026-06-11
description: "Shader Showcase 后处理特效系列 — 锐化效果 (Sharpen)详解，包含算法原理、参数说明与完整注释源码"
tags: ["Shader", "GLSL", "后处理", "锐化", "图像处理"]
category: "Shader分析"
draft: false
---

> 本文是 **Shader Showcase 后处理特效着色器源码详解** 系列的第 4 篇。

### 效果概述

锐化效果用于增强图像中的边缘和细节，属于 **图像增强** 类别。本实现采用经典的 **Unsharp Mask（USM）** 算法：先对图像进行高斯模糊得到低频分量，然后用原图减去模糊图得到高频细节，最后将高频细节按强度叠加回原图。这种方法能有效增强边缘而不会引入过多噪声。

### 参数说明

| 参数名 | UI 标签 | 类型 | 最小值 | 最大值 | 默认值 | UI 控件 |
|--------|---------|------|--------|--------|--------|---------|
| `uParamFloat0` | 强度 | Float | 0.0 | 3.0 | 1.0 | 滑块 |
| `uParamFloat1` | 半径 | Float | 0.5 | 3.0 | 1.0 | 滑块 |

- **强度**：控制锐化程度，值越大边缘增强越明显
- **半径**：控制用于计算模糊的采样范围，影响锐化的"粗细"

### 算法原理

**Unsharp Mask 公式：**

$$
\text{sharpened} = \text{original} + \text{Amount} \times (\text{original} - \text{blur})
$$

其中 `blur` 是对原图进行高斯模糊的结果。`original - blur` 提取了图像的高频分量（边缘和细节），乘以 `Amount` 后叠加回原图即完成锐化。

**高斯模糊核：**

与泛光效果类似，使用动态范围的高斯核：

$$
w(x, y) = e^{-\frac{x^2 + y^2}{2\sigma^2}}, \quad \sigma = \text{Radius} \times 0.5
$$

采样范围为 `[-ceil(Radius), ceil(Radius)]`，偏移量乘以 `Radius` 以控制扩散。

最终结果通过 `clamp()` 限制在 `[0, 1]` 范围内，防止过曝。

### 逐段代码分析

**参数读取与提前退出：**

```glsl
void main() {
    vec3 color = texture(uInputTex, vUV).rgb;

    float Amount = uParamFloat0;
    float Radius = uParamFloat1;

    // 提前退出：锐化强度为 0 时直接输出原图
    if (Amount <= 0.0) {
        outColor = vec4(color, 1.0);
        return;
    }
```

**高斯模糊计算（用于提取低频分量）：**

使用动态范围的高斯核，半径由参数控制。

```glsl
    // 高斯加权模糊，用于 Unsharp Mask
    vec2 texelSize = 1.0 / uResolution;
    float sigma = Radius * 0.5;
    float sigma2 = 2.0 * sigma * sigma;
    float total = 0.0;
    vec3 blur = vec3(0.0);

    // 动态范围：由 Radius 参数决定
    int range = int(ceil(Radius));
    for (int x = -range; x <= range; x++) {
        for (int y = -range; y <= range; y++) {
            float dist2 = float(x * x + y * y);
            float w = exp(-dist2 / sigma2);
            vec2 offset = vec2(float(x), float(y)) * texelSize * Radius;
            blur += texture(uInputTex, vUV + offset).rgb * w;
            total += w;
        }
    }
    blur /= max(total, 0.001);
```

**Unsharp Mask 锐化与输出：**

```glsl
    // Unsharp Mask：原图 + 强度 * (原图 - 模糊)
    vec3 sharpened = color + (color - blur) * Amount;

    // 限制输出范围防止过曝
    outColor = vec4(clamp(sharpened, 0.0, 1.0), 1.0);
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
    float uParamFloat0;   // 锐化强度
    float uParamFloat1;   // 锐化半径
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

    // 读取锐化参数
    float Amount = uParamFloat0;   // 锐化强度
    float Radius = uParamFloat1;    // 锐化半径

    // 提前退出：锐化强度为 0 时直接输出原图
    if (Amount <= 0.0) {
        outColor = vec4(color, 1.0);
        return;
    }

    // 计算纹素大小
    vec2 texelSize = 1.0 / uResolution;
    // 高斯 sigma 与半径成正比
    float sigma = Radius * 0.5;
    float sigma2 = 2.0 * sigma * sigma;
    float total = 0.0;
    vec3 blur = vec3(0.0);

    // 动态范围的高斯模糊核
    int range = int(ceil(Radius));
    for (int x = -range; x <= range; x++) {
        for (int y = -range; y <= range; y++) {
            // 距离平方
            float dist2 = float(x * x + y * y);
            // 高斯权重
            float w = exp(-dist2 / sigma2);
            // 采样偏移量乘以半径
            vec2 offset = vec2(float(x), float(y)) * texelSize * Radius;
            blur += texture(uInputTex, vUV + offset).rgb * w;
            total += w;
        }
    }
    // 归一化模糊结果
    blur /= max(total, 0.001);

    // Unsharp Mask 锐化公式
    // sharpened = original + amount * (original - blur)
    // (original - blur) 提取高频细节
    vec3 sharpened = color + (color - blur) * Amount;

    // clamp 到 [0,1] 防止颜色溢出
    outColor = vec4(clamp(sharpened, 0.0, 1.0), 1.0);
}
```

---