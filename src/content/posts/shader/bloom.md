---
title: "泛光效果 (Bloom)"
published: 2026-06-11
description: "Shader Showcase 后处理特效系列 — 泛光效果 (Bloom)详解，包含算法原理、参数说明与完整注释源码"
tags: ["Shader", "GLSL", "后处理", "泛光", "光照"]
category: "Shader分析"
draft: false
---

> 本文是 **Shader Showcase 后处理特效着色器源码详解** 系列的第 2 篇。

### 效果概述

泛光（Bloom）是一种模拟真实相机镜头光晕的后期处理特效，属于 **光照** 类别。它通过提取画面中亮度超过阈值的区域，对这些高亮区域进行高斯模糊处理，最后将模糊后的光晕叠加回原始图像，产生一种柔和的发光效果。常用于增强画面中光源、高光反射等明亮区域的视觉冲击力。

### 参数说明

| 参数名 | UI 标签 | 类型 | 最小值 | 最大值 | 默认值 | UI 控件 |
|--------|---------|------|--------|--------|--------|---------|
| `uParamFloat0` | 泛光强度 | Float | 0.0 | 3.0 | 0.8 | 滑块 |
| `uParamFloat1` | 阈值 | Float | 0.0 | 1.0 | 0.7 | 滑块 |
| `uParamFloat2` | 模糊大小 | Float | 1.0 | 10.0 | 4.0 | 滑块 |

- **泛光强度**：控制最终叠加的泛光亮度倍率
- **阈值**：只有亮度超过此值的像素才会参与泛光计算
- **模糊大小**：控制高斯模糊核的采样范围和扩散程度

### 算法原理

泛光效果采用 **单通道实现**，核心步骤如下：

**1. 亮度阈值提取：**

对每个采样点计算其 ITU-R BT.709 亮度值：

$$
L = 0.2126 \cdot R + 0.7152 \cdot G + 0.0722 \cdot B
$$

只有 $L > \text{Threshold}$ 的像素才参与泛光，超出部分按比例缩放：

$$
\text{brightness} = \frac{L - T}{1 - T + 0.001}
$$

**2. 二维高斯模糊：**

使用二维高斯核进行卷积，权重为：

$$
w(x, y) = e^{-\frac{x^2 + y^2}{2\sigma^2}}
$$

其中 $\sigma = \text{BlurSize}$，采样范围由 `range = ceil(BlurSize)` 决定。

**3. 加法混合：**

$$
\text{final} = \text{original} + \text{bloom} \times \text{BloomIntensity}
$$

### 逐段代码分析

**参数读取与提前退出：**

从 UBO 读取三个控制参数，当泛光强度为 0 时直接输出原图，避免不必要的计算。

```glsl
void main() {
    vec3 color = texture(uInputTex, vUV).rgb;

    float BloomIntensity = uParamFloat0;
    float Threshold = uParamFloat1;
    float BlurSize = max(uParamFloat2, 1.0);

    // 提前退出：无泛光时直接输出原图
    if (BloomIntensity <= 0.0) {
        outColor = vec4(color, 1.0);
        return;
    }
```

**高斯模糊核计算：**

计算纹素大小、模糊范围和 sigma 值，准备二维高斯卷积。

```glsl
    vec2 texelSize = 1.0 / uResolution;
    
    // BlurSize 同时控制核范围和采样偏移量，产生明显的视觉效果
    int range = int(ceil(BlurSize));
    float sigma = BlurSize;
    float sigma2 = 2.0 * sigma * sigma;
```

**二维高斯卷积循环：**

遍历以当前像素为中心的方形区域，对每个采样点计算高斯权重，并根据亮度阈值过滤。

```glsl
    vec3 bloom = vec3(0.0);
    float weightSum = 0.0;

    for (int x = -range; x <= range; x++) {
        for (int y = -range; y <= range; y++) {
            // 计算高斯权重
            float dist2 = float(x * x + y * y);
            float w = exp(-dist2 / sigma2);
            
            // 采样偏移量乘以 BlurSize，扩大模糊范围
            vec2 offset = vec2(float(x), float(y)) * texelSize * BlurSize;
            vec3 sampleColor = texture(uInputTex, vUV + offset).rgb;
            
            // 阈值过滤：只有亮度超过阈值的像素才参与泛光
            float sampleLuma = dot(sampleColor, vec3(0.2126, 0.7152, 0.0722));
            if (sampleLuma > Threshold) {
                // 超出阈值的亮度按比例贡献
                float brightness = (sampleLuma - Threshold) / (1.0 - Threshold + 0.001);
                bloom += sampleColor * brightness * w;
            }
            weightSum += w;
        }
    }
    bloom /= max(weightSum, 0.001);
```

**最终混合输出：**

将泛光结果按强度叠加到原始图像上。

```glsl
    outColor = vec4(color + bloom * BloomIntensity, 1.0);
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
    float uParamFloat0;   // 泛光强度
    float uParamFloat1;   // 亮度阈值
    float uParamFloat2;   // 模糊大小
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

    // 读取控制参数
    float BloomIntensity = uParamFloat0;  // 泛光强度
    float Threshold = uParamFloat1;       // 亮度阈值
    float BlurSize = max(uParamFloat2, 1.0); // 模糊大小，最小为 1.0

    // 提前退出：泛光强度为 0 时直接输出原图
    if (BloomIntensity <= 0.0) {
        outColor = vec4(color, 1.0);
        return;
    }

    // 计算单个纹素的大小（用于纹理坐标偏移）
    vec2 texelSize = 1.0 / uResolution;
    
    // 模糊范围 = ceil(BlurSize)，决定卷积核的半径
    int range = int(ceil(BlurSize));
    // sigma 直接等于 BlurSize
    float sigma = BlurSize;
    // 2 * sigma^2，用于高斯公式
    float sigma2 = 2.0 * sigma * sigma;
    
    // 泛光累积值和权重总和
    vec3 bloom = vec3(0.0);
    float weightSum = 0.0;

    // 二维高斯卷积：遍历 [-range, range] 的方形区域
    for (int x = -range; x <= range; x++) {
        for (int y = -range; y <= range; y++) {
            // 计算采样点到中心距离的平方
            float dist2 = float(x * x + y * y);
            // 高斯权重：w = exp(-dist^2 / (2*sigma^2))
            float w = exp(-dist2 / sigma2);
            
            // 采样偏移 = 整数偏移 * 纹素大小 * BlurSize
            // 乘以 BlurSize 进一步扩大模糊扩散范围
            vec2 offset = vec2(float(x), float(y)) * texelSize * BlurSize;
            vec3 sampleColor = texture(uInputTex, vUV + offset).rgb;
            
            // 使用 BT.709 标准计算采样点亮度
            float sampleLuma = dot(sampleColor, vec3(0.2126, 0.7152, 0.0722));
            // 阈值过滤：只有亮度超过阈值的像素才产生泛光
            if (sampleLuma > Threshold) {
                // 超出阈值部分按比例贡献亮度
                float brightness = (sampleLuma - Threshold) / (1.0 - Threshold + 0.001);
                bloom += sampleColor * brightness * w;
            }
            weightSum += w;
        }
    }
    // 归一化泛光结果
    bloom /= max(weightSum, 0.001);

    // 将泛光按强度叠加到原始图像
    outColor = vec4(color + bloom * BloomIntensity, 1.0);
}
```

---