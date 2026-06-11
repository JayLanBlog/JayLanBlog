---
title: "故障艺术 (Glitch Art)"
published: 2026-06-11
description: "Shader Showcase 后处理特效系列 — 故障艺术 (Glitch Art)详解，包含算法原理、参数说明与完整注释源码"
tags: ["Shader", "GLSL", "后处理", "故障", "艺术效果"]
category: "Shader分析"
draft: false
---

> 本文是 **Shader Showcase 后处理特效着色器源码详解** 系列的第 13 篇。

### 效果概述

故障艺术着色器属于 **Stylize（风格化）** 类别，模拟了数字信号故障产生的视觉失真效果。该着色器综合了三种经典的故障表现形态：扫描线撕裂（Scanline Tearing）、RGB 通道分离（Chromatic Aberration）和随机色块污染（Block Artifacts）。这三种效果以随机概率触发，故障强度参数控制各效果的发生概率和幅度。该效果广泛应用于赛博朋克风格、音乐视频、科幻电影和数字艺术作品中。

### 参数说明

| 参数名 | 标签 | 类型 | 最小值 | 最大值 | 默认值 | UI 类型 | 说明 |
|--------|------|------|--------|--------|--------|---------|------|
| `uParamFloat0` | 故障强度 | Float | 0.0 | 1.0 | 0.5 | slider | 控制所有故障效果的发生概率和幅度 |
| `uParamFloat1` | 速度 | Float | 0.0 | 10.0 | 3.0 | slider | 故障效果的切换频率 |
| `uParamFloat2` | 块大小 | Float | 1.0 | 50.0 | 10.0 | slider | 扫描线撕裂和色块的基本单元大小（像素） |

### 算法原理

#### 1. 伪随机数生成

使用经典的 GLSL 伪随机数生成公式：

$$\text{random}(\mathbf{st}) = \text{fract}(\sin(\text{dot}(\mathbf{st}, (12.9898, 78.233))) \times 43758.5453)$$

该公式通过正弦函数的大数乘法和取小数运算，将二维坐标映射为伪随机值。

#### 2. 扫描线撕裂（Scanline Tearing）

以时间帧为单位，随机选择某些水平扫描线进行水平偏移：

$$P(\text{tear}) = \text{step}(0.95 - g \times 0.5, \text{random}(y_{line}, t))$$

$$\text{offset} = (\text{random}(t, y_{50}) - 0.5) \times g \times 0.3$$

其中 $g$ 为故障强度，$t$ 为离散化时间，$y_{line}$ 为像素所在的扫描线编号。当随机值超过阈值时，该扫描线上的所有像素在水平方向发生偏移。

#### 3. RGB 通道分离（Chromatic Aberration）

以较低概率触发 RGB 三通道的水平偏移，模拟色差效果：

$$P(\text{split}) = \text{step}(0.98 - g \times 0.3, \text{random}(t, 1.0))$$

$$\Delta x = P(\text{split}) \times g \times 0.05$$

红色通道向右偏移 $\Delta x$，蓝色通道向左偏移 $\Delta x$，绿色通道保持不变。

#### 4. 随机色块（Block Artifacts）

将画面划分为网格块，随机选择某些块替换为纯色噪声：

$$P(\text{block}) = \text{step}(0.99 - g \times 0.2, \text{random}(x_{block}, y_{block} + t))$$

被选中的块颜色为基于时间和垂直位置的随机值。

### 逐段代码分析

#### 段落一：着色器声明与伪随机函数

标准声明结构加上经典的 GLSL 伪随机数生成函数。该随机函数使用正弦函数的点积输入和大数乘法，产生视觉上足够随机的值。

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

#### 段落二：主函数 — 扫描线撕裂

主函数首先对时间进行离散化（`floor`），使故障效果以帧为单位跳变而非连续变化。然后通过随机概率决定每条扫描线是否发生水平偏移，偏移量也由随机函数决定。

```glsl
void main() {
    vec3 color = texture(uInputTex, vUV).rgb;
    float t = floor(uTime * uParamFloat1); // 离散化时间

    // 随机行偏移 (扫描线撕裂)
    float lineNoise = step(0.95 - uParamFloat0 * 0.5, random(vec2(floor(vUV.y * uResolution.y / uParamFloat2), t)));
    float offset = (random(vec2(t, floor(vUV.y * 50.0))) - 0.5) * uParamFloat0 * 0.3;
    color = texture(uInputTex, vUV + vec2(offset, 0.0)).rgb;
```

#### 段落三：RGB 通道分离与随机色块

RGB 通道分离以较低概率触发，红蓝通道向相反方向偏移。随机色块将画面划分为网格，随机选择某些块替换为纯色噪声值。

```glsl
    // RGB 通道分离
    float channelShift = step(0.98 - uParamFloat0 * 0.3, random(vec2(t, 1.0))) * uParamFloat0 * 0.05;
    color.r = texture(uInputTex, vUV + vec2(channelShift, 0.0)).r;
    color.b = texture(uInputTex, vUV - vec2(channelShift, 0.0)).b;

    // 随机色块
    float blockNoise = step(0.99 - uParamFloat0 * 0.2, random(vec2(floor(vUV.x * uResolution.x / uParamFloat2), floor(vUV.y * uResolution.y / uParamFloat2) + t)));
    color = mix(color, vec3(random(vec2(t, vUV.y * 100.0))), blockNoise);

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
    float uParamFloat0;  // 故障强度
    float uParamFloat1;  // 速度
    float uParamFloat2;  // 块大小
    float uParamFloat3;  // 保留参数
    float uParamFloat4;  // 保留参数
    float uParamFloat5;  // 保留参数
    vec2 uResolution;    // 屏幕分辨率
    float uTime;         // 时间
    float uFrameCount;   // 帧计数
};

// 经典GLSL伪随机数生成函数
// 通过正弦函数的大数乘法和取小数运算产生伪随机值
float random(vec2 st) {
    return fract(sin(dot(st, vec2(12.9898, 78.233))) * 43758.5453);
}

void main() {
    // 采样原始颜色
    vec3 color = texture(uInputTex, vUV).rgb;
    // 将时间离散化为帧索引，使故障效果以帧为单位跳变
    float t = floor(uTime * uParamFloat1);

    // === 扫描线撕裂 ===
    // 以块大小为单位划分扫描线，随机决定是否发生偏移
    float lineNoise = step(0.95 - uParamFloat0 * 0.5, random(vec2(floor(vUV.y * uResolution.y / uParamFloat2), t)));
    // 偏移量由随机函数决定，方向和大小都随机
    float offset = (random(vec2(t, floor(vUV.y * 50.0))) - 0.5) * uParamFloat0 * 0.3;
    // 使用偏移后的UV重新采样
    color = texture(uInputTex, vUV + vec2(offset, 0.0)).rgb;

    // === RGB通道分离 ===
    // 以较低概率触发色差效果
    float channelShift = step(0.98 - uParamFloat0 * 0.3, random(vec2(t, 1.0))) * uParamFloat0 * 0.05;
    // 红色通道向右偏移，蓝色通道向左偏移
    color.r = texture(uInputTex, vUV + vec2(channelShift, 0.0)).r;
    color.b = texture(uInputTex, vUV - vec2(channelShift, 0.0)).b;

    // === 随机色块 ===
    // 将画面划分为网格块，随机选择某些块替换为纯色噪声
    float blockNoise = step(0.99 - uParamFloat0 * 0.2, random(vec2(floor(vUV.x * uResolution.x / uParamFloat2), floor(vUV.y * uResolution.y / uParamFloat2) + t)));
    // 将选中块的颜色替换为随机纯色
    color = mix(color, vec3(random(vec2(t, vUV.y * 100.0))), blockNoise);

    outColor = vec4(color, 1.0);
}
```

---