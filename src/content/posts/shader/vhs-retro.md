---
title: "VHS 复古效果 (VHS Retro)"
published: 2026-06-11
description: "Shader Showcase 后处理特效系列 — VHS 复古效果 (VHS Retro)详解，包含算法原理、参数说明与完整注释源码"
tags: ["Shader", "GLSL", "后处理", "复古", "VHS", "艺术效果"]
category: "Shader分析"
draft: false
---

> 本文是 **Shader Showcase 后处理特效着色器源码详解** 系列的第 15 篇。

### 效果概述

VHS 复古着色器属于 **Retro（复古怀旧）** 类别，模拟了模拟录像带（VHS）的典型视觉特征。该着色器综合了四种 VHS 特征效果：水平跟踪失真（Tracking Distortion）模拟录像带播放时偶尔出现的水平跳动；色彩漂移（Color Drift）模拟磁头对齐不良导致的 Y/C 信号延迟；扫描线（Scanlines）模拟 CRT 电视的行扫描结构；随机噪声（Noise）模拟磁带信号劣化产生的雪花点。此外还叠加了轻微的色彩偏移（偏蓝/偏红），还原 VHS 录像的独特色彩质感。

### 参数说明

| 参数名 | 标签 | 类型 | 最小值 | 最大值 | 默认值 | UI 类型 | 说明 |
|--------|------|------|--------|--------|--------|---------|------|
| `uParamFloat0` | 扫描线强度 | Float | 0.0 | 1.0 | 0.3 | slider | 扫描线暗化程度 |
| `uParamFloat1` | 噪声量 | Float | 0.0 | 0.5 | 0.1 | slider | 随机雪花噪声强度 |
| `uParamFloat2` | 色彩漂移 | Float | 0.0 | 0.02 | 0.005 | slider | RGB 通道水平偏移量 |
| `uParamFloat3` | 跟踪失真 | Float | 0.0 | 0.1 | 0.02 | slider | 水平跟踪失真的发生概率 |

### 算法原理

#### 1. 水平跟踪失真（Tracking Distortion）

VHS 录像带播放时，磁头跟踪不良会导致整帧画面水平跳动。该效果通过随机概率触发：

$$P(\text{tracking}) = \text{step}(0.98 - g, \text{random}(\lfloor t \times 10 \rfloor, 0))$$

$$\Delta x = P(\text{tracking}) \times (\text{random}(t, 1) - 0.5) \times 0.1$$

其中 $g$ 为跟踪失真参数，$t$ 为时间。当触发时，整帧画面在水平方向发生随机偏移。

#### 2. 色彩漂移（Color Drift / Chroma Delay）

VHS 信号的亮度（Y）和色度（C）分量通过不同时间处理，磁头对齐不良会导致色度信号延迟。该效果通过正弦波驱动的 RGB 通道水平偏移模拟：

$$\Delta x(y) = \sin(y \times \frac{H}{2} + t \times 2) \times d$$

其中 $H$ 为屏幕高度（像素），$d$ 为漂移参数。红色通道向右偏移 $\Delta x$，蓝色通道向左偏移 $\Delta x$，绿色通道保持不变。

#### 3. 扫描线（Scanlines）

CRT 电视的电子束逐行扫描，行间有微小间隙导致亮度变化：

$$\text{scanline}(y) = \sin(y \times H \times \pi) \times 0.5 + 0.5$$

$$C_{scan} = C \times (1 - \text{scanline} \times s)$$

其中 $s$ 为扫描线强度参数。

#### 4. 随机噪声（Noise）

模拟磁带信号劣化产生的雪花点：

$$C_{noise} = C_{scan} + (\text{random}(\text{uv} \times \text{res} + t \times 100) - 0.5) \times n$$

其中 $n$ 为噪声量参数。

#### 5. VHS 色彩偏移

VHS 录像通常带有轻微的色彩偏差，通过简单的通道乘法实现：

$$C_{final}.r = C_{noise}.r \times 1.05, \quad C_{final}.b = C_{noise}.b \times 0.95$$

### 逐段代码分析

#### 段落一：着色器声明与伪随机函数

标准声明结构加上伪随机函数。VHS 效果中多处需要随机值来驱动故障和噪声。

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

float random(vec2 st) {
    return fract(sin(dot(st, vec2(12.9898, 78.233))) * 43758.5453);
}
```

#### 段落二：跟踪失真与色彩漂移

跟踪失真以低概率触发整帧水平偏移，模拟 VHS 磁头跟踪不良。色彩漂移通过正弦波驱动 RGB 通道的水平偏移，偏移量随垂直位置变化，模拟磁头对齐偏差。

```glsl
void main() {
    // 水平跟踪失真
    float trackingLine = step(0.98 - uParamFloat3, random(vec2(floor(uTime * 10.0), 0.0)));
    float trackingOffset = trackingLine * (random(vec2(uTime, 1.0)) - 0.5) * 0.1;
    vec2 uv = vUV + vec2(trackingOffset, 0.0);

    // 色彩漂移 (Y通道延迟)
    float drift = sin(uv.y * uResolution.y * 0.5 + uTime * 2.0) * uParamFloat2;
    float r = texture(uInputTex, uv + vec2(drift, 0.0)).r;
    float g = texture(uInputTex, uv).g;
    float b = texture(uInputTex, uv - vec2(drift, 0.0)).b;
    vec3 color = vec3(r, g, b);
```

#### 段落三：扫描线、噪声与色彩偏移

扫描线通过正弦函数产生周期性明暗变化。噪声通过随机函数叠加雪花点。最后进行轻微的色彩偏移（红色增强、蓝色减弱），模拟 VHS 录像的独特色彩特征。

```glsl
    // 扫描线
    float scanline = sin(uv.y * uResolution.y * 3.14159) * 0.5 + 0.5;
    color *= 1.0 - scanline * uParamFloat0;

    // 噪声
    float noise = random(vec2(uv * uResolution + uTime * 100.0));
    color += (noise - 0.5) * uParamFloat1;

    // VHS 色彩偏移 (轻微偏蓝/偏红)
    color.r *= 1.05;
    color.b *= 0.95;

    outColor = vec4(clamp(color, 0.0, 1.0), 1.0);
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
    float uParamFloat0;  // 扫描线强度
    float uParamFloat1;  // 噪声量
    float uParamFloat2;  // 色彩漂移
    float uParamFloat3;  // 跟踪失真
    float uParamFloat4;  // 保留参数
    float uParamFloat5;  // 保留参数
    vec2 uResolution;    // 屏幕分辨率
    float uTime;         // 时间
    float uFrameCount;   // 帧计数
};

// 伪随机数生成函数
float random(vec2 st) {
    return fract(sin(dot(st, vec2(12.9898, 78.233))) * 43758.5453);
}

void main() {
    // === 水平跟踪失真 ===
    // 以低概率触发整帧水平偏移，模拟VHS磁头跟踪不良
    float trackingLine = step(0.98 - uParamFloat3, random(vec2(floor(uTime * 10.0), 0.0)));
    float trackingOffset = trackingLine * (random(vec2(uTime, 1.0)) - 0.5) * 0.1;
    vec2 uv = vUV + vec2(trackingOffset, 0.0);

    // === 色彩漂移 (Y/C延迟) ===
    // 通过正弦波驱动RGB通道的水平偏移，模拟磁头对齐偏差
    float drift = sin(uv.y * uResolution.y * 0.5 + uTime * 2.0) * uParamFloat2;
    float r = texture(uInputTex, uv + vec2(drift, 0.0)).r;  // 红色通道右偏
    float g = texture(uInputTex, uv).g;                      // 绿色通道不变
    float b = texture(uInputTex, uv - vec2(drift, 0.0)).b;  // 蓝色通道左偏
    vec3 color = vec3(r, g, b);

    // === 扫描线 ===
    // 通过正弦函数产生周期性明暗变化，模拟CRT行扫描结构
    float scanline = sin(uv.y * uResolution.y * 3.14159) * 0.5 + 0.5;
    color *= 1.0 - scanline * uParamFloat0;

    // === 随机噪声 ===
    // 叠加基于位置和时间的随机噪声，模拟磁带信号劣化
    float noise = random(vec2(uv * uResolution + uTime * 100.0));
    color += (noise - 0.5) * uParamFloat1;

    // === VHS色彩偏移 ===
    // 轻微增强红色、减弱蓝色，还原VHS录像的色彩特征
    color.r *= 1.05;
    color.b *= 0.95;

    // 输出最终颜色，限制到[0,1]范围
    outColor = vec4(clamp(color, 0.0, 1.0), 1.0);
}
```

---