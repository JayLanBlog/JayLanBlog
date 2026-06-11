---
title: "灰度测试 (Grayscale Test)"
published: 2026-06-11
description: "Shader Showcase 后处理特效系列 — 灰度测试 (Grayscale Test)详解，包含算法原理、参数说明与完整注释源码"
tags: ["Shader", "GLSL", "后处理", "灰度"]
category: "Shader分析"
draft: false
---

> 本文是 **Shader Showcase 后处理特效着色器源码详解** 系列的第 1 篇。

### 效果概述

灰度测试是最基础的后处理特效，用于验证 Shader 管线是否正常工作。它将彩色图像转换为灰度图像，属于 **基础颜色处理** 类别。该效果通过 ITU-R BT.601 标准的亮度加权公式计算灰度值，并提供强度参数控制灰度化程度。

### 参数说明

| 参数名 | UI 标签 | 类型 | 最小值 | 最大值 | 默认值 | UI 控件 |
|--------|---------|------|--------|--------|--------|---------|
| `uParamFloat0` | 灰度强度 | Float | 0.0 | 1.0 | 0.5 | 滑块 |

- **灰度强度**：控制灰度化程度。0.0 时输出全黑，1.0 时输出标准灰度，0.5 时输出半亮度灰度。

### 算法原理

灰度转换的核心是 **ITU-R BT.601 亮度公式**。人眼对绿色最敏感，对蓝色最不敏感，因此三个通道的权重不同：

$$
L = 0.299 \cdot R + 0.587 \cdot G + 0.114 \cdot B
$$

在 GLSL 中，使用 `dot()` 点积运算高效计算：

$$
\text{gray} = \text{dot}(\text{color}, \vec{3}(0.299, 0.587, 0.114)) \times \text{uParamFloat0}
$$

最终将灰度值复制到 RGB 三个通道，Alpha 设为 1.0。

### 逐段代码分析

**片段着色器声明与资源绑定：**

声明输入 UV、输出颜色、输入纹理和 UBO 参数块。这是所有后处理着色器的标准模板。

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

**主函数 — 灰度计算：**

采样输入纹理，使用 BT.601 权重计算灰度值，乘以强度参数后输出。

```glsl
void main() {
    // 采样输入纹理获取原始颜色
    vec3 color = texture(uInputTex, vUV).rgb;
    // 使用 BT.601 亮度权重计算灰度值，乘以强度参数
    float gray = dot(color, vec3(0.299, 0.587, 0.114)) * uParamFloat0;
    // 将灰度值写入 RGB 三通道，Alpha 设为 1.0
    outColor = vec4(vec3(gray), 1.0);
}
```

### 完整源码

```glsl
#version 460
// 输入：从顶点着色器插值得到的纹理坐标
layout(location=0) in vec2 vUV;
// 输出：最终像素颜色
layout(location=0) out vec4 outColor;

// 输入纹理绑定到 binding=0
layout(binding=0) uniform sampler2D uInputTex;

// 统一参数块，std140 标准布局，绑定到 binding=1
layout(std140, binding=1) uniform Params {
    float uParamFloat0;   // 灰度强度
    float uParamFloat1;   // 未使用
    float uParamFloat2;   // 未使用
    float uParamFloat3;   // 未使用
    float uParamFloat4;   // 未使用
    float uParamFloat5;   // 未使用
    vec2  uResolution;    // 屏幕分辨率
    float uTime;          // 运行时间
    float uFrameCount;    // 帧计数
};

void main() {
    // 从输入纹理采样当前像素的颜色
    vec3 color = texture(uInputTex, vUV).rgb;
    // 使用 ITU-R BT.601 标准权重进行灰度转换
    // 人眼对绿色最敏感(0.587)，红色次之(0.299)，蓝色最不敏感(0.114)
    float gray = dot(color, vec3(0.299, 0.587, 0.114)) * uParamFloat0;
    // 输出灰度图像，RGB 三通道相同，Alpha 通道为 1.0
    outColor = vec4(vec3(gray), 1.0);
}
```

---