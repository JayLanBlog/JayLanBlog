---
title: "暗角效果 (Vignette)"
published: 2026-06-11
description: "Shader Showcase 后处理特效系列 — 暗角效果 (Vignette)详解，包含算法原理、参数说明与完整注释源码"
tags: ["Shader", "GLSL", "后处理", "暗角", "镜头效果"]
category: "Shader分析"
draft: false
---

> 本文是 **Shader Showcase 后处理特效着色器源码详解** 系列的第 8 篇。

### 效果概述

暗角效果模拟真实相机镜头的光学衰减现象，使图像边缘逐渐变暗，属于 **颜色调整** 类别。这种效果能自然地将观者注意力引导到画面中心，同时增添电影感和专业摄影质感。本实现支持调节暗角强度、柔和度和圆度三个参数，并自动进行宽高比校正。

### 参数说明

| 参数名 | UI 标签 | 类型 | 最小值 | 最大值 | 默认值 | UI 控件 |
|--------|---------|------|--------|--------|--------|---------|
| `uParamFloat0` | 暗角强度 | Float | 0.0 | 2.0 | 0.5 | 滑块 |
| `uParamFloat1` | 柔和度 | Float | 0.0 | 1.0 | 0.5 | 滑块 |
| `uParamFloat2` | 圆度 | Float | 0.0 | 1.0 | 0.5 | 滑块 |

- **暗角强度**：控制暗角的衰减程度，值越大边缘越暗
- **柔和度**：控制暗角过渡的平滑程度，值越大过渡越柔和
- **圆度**：控制暗角形状的圆度，0 为椭圆形（不校正宽高比），1 为正圆形（校正宽高比）

### 算法原理

**暗角计算步骤：**

1. 将 UV 坐标以画面中心为原点，进行宽高比校正：

$$
\vec{uv}_{\text{centered}} = \vec{uv} - 0.5, \quad uv_x \leftarrow uv_x \times \text{mix}(1, \text{aspect}, \text{roundness})
$$

2. 计算到中心的距离：

$$
d = |\vec{uv}_{\text{centered}}|
$$

3. 使用 `smoothstep` 生成平滑衰减：

$$
\text{vignette} = \text{smoothstep}(0.5 - S \times 0.5,\; 0.5 + S \times 0.2,\; d \times (1 + I))
$$

其中 $I$ 为暗角强度，$S$ 为柔和度。

4. 最终颜色：

$$
\text{output} = \text{color} \times (1 - \text{vignette})
$$

`smoothstep` 函数在两个边界之间产生平滑的 0→1 过渡，使暗角效果自然柔和。

### 逐段代码分析

**坐标中心化与宽高比校正：**

```glsl
void main() {
    vec3 color = texture(uInputTex, vUV).rgb;
    // 将 UV 坐标以画面中心为原点
    vec2 uv = vUV - 0.5;
    // 宽高比校正：圆度参数控制校正程度
    float aspect = uResolution.x / uResolution.y;
    uv.x *= mix(1.0, aspect, uParamFloat2);
```

**暗角衰减计算：**

```glsl
    // 计算到中心的欧几里得距离
    float dist = length(uv);
    // smoothstep 生成平滑衰减曲线
    // 第一个参数：开始衰减的距离（受柔和度影响）
    // 第二个参数：完全衰减的距离（受柔和度影响）
    // 第三个参数：当前像素距离（受强度缩放）
    float vignette = smoothstep(0.5 - uParamFloat1 * 0.5, 0.5 + uParamFloat1 * 0.2, dist * (1.0 + uParamFloat0));
    // 将暗角衰减乘到颜色上
    color *= 1.0 - vignette;
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
    float uParamFloat0;   // 暗角强度
    float uParamFloat1;   // 柔和度
    float uParamFloat2;   // 圆度
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
    // 将 UV 坐标以画面中心 (0.5, 0.5) 为原点
    vec2 uv = vUV - 0.5;
    // 宽高比校正：圆度参数控制 x 方向的缩放
    // 圆度=0 时不校正（椭圆形暗角），圆度=1 时完全校正（正圆形暗角）
    float aspect = uResolution.x / uResolution.y;
    uv.x *= mix(1.0, aspect, uParamFloat2);
    // 计算当前像素到中心的欧几里得距离
    float dist = length(uv);
    // 使用 smoothstep 生成平滑的暗角衰减
    // 参数1 (edge0): 衰减起始距离 = 0.5 - 柔和度*0.5
    // 参数2 (edge1): 衰减结束距离 = 0.5 + 柔和度*0.2
    // 参数3 (x): 当前距离 * (1 + 强度)，强度越大有效距离越大，暗角越强
    float vignette = smoothstep(0.5 - uParamFloat1 * 0.5, 0.5 + uParamFloat1 * 0.2, dist * (1.0 + uParamFloat0));
    // 将暗角衰减应用到颜色上（1 - vignette 使边缘变暗）
    color *= 1.0 - vignette;
    outColor = vec4(color, 1.0);
}
```

---