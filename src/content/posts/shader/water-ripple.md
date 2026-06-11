---
title: "水波纹效果 (Water Ripple)"
published: 2026-06-11
description: "Shader Showcase 后处理特效系列 — 水波纹效果 (Water Ripple)详解，包含算法原理、参数说明与完整注释源码"
tags: ["Shader", "GLSL", "后处理", "水波纹", "模拟"]
category: "Shader分析"
draft: false
---

> 本文是 **Shader Showcase 后处理特效着色器源码详解** 系列的第 17 篇。

### 效果概述

水波纹着色器属于 **Distort（扭曲变形）** 类别，模拟了水面波纹的光学折射效果。该着色器通过三层不同方向、频率和速度的正弦波叠加，生成复杂的水面波纹图案，并基于波纹的法线近似计算折射偏移，使背景图像产生类似透过水面观看的扭曲效果。此外还叠加了高光反射（模拟水面光泽）和深度明暗变化（波峰亮、波谷暗），增强水面的真实感。该效果可应用于水面反射模拟、玻璃折射、热空气扰动等场景。

### 参数说明

| 参数名 | 标签 | 类型 | 最小值 | 最大值 | 默认值 | UI 类型 | 说明 |
|--------|------|------|--------|--------|--------|---------|------|
| `uParamFloat0` | 振幅 | Float | 0.0 | 0.05 | 0.01 | slider | 波纹的振幅高度 |
| `uParamFloat1` | 频率 | Float | 1.0 | 50.0 | 15.0 | slider | 波纹的空间频率 |
| `uParamFloat2` | 速度 | Float | 0.0 | 5.0 | 2.0 | slider | 波纹的传播速度 |
| `uParamFloat3` | 折射强度 | Float | 0.0 | 0.1 | 0.02 | slider | 折射偏移的强度 |

### 算法原理

#### 1. 多层正弦波叠加

水面波纹由三个不同方向的正弦波叠加而成：

$$h_1(x, t) = A \cdot \sin(f \cdot x + \omega \cdot t)$$

$$h_2(y, t) = 0.7A \cdot \sin(0.8f \cdot y + 1.3\omega \cdot t)$$

$$h_3(x, y, t) = 0.5A \cdot \sin(0.5f \cdot (x + y) + 0.7\omega \cdot t)$$

其中 $A$ 为振幅，$f$ 为频率，$\omega$ 为角速度。三个波分别沿水平、垂直和对角方向传播，频率和速度各不相同，叠加后产生复杂的干涉图案。

#### 2. 法线近似（Normal Approximation）

水面法线通过波高函数的偏导数近似计算：

$$n_x = \frac{\partial h}{\partial x} \approx \frac{\partial h_1}{\partial x} + \frac{\partial h_3}{\partial x}$$

$$n_y = \frac{\partial h}{\partial y} \approx \frac{\partial h_2}{\partial y} + \frac{\partial h_3}{\partial y}$$

正弦函数的导数为余弦函数，因此：

$$\frac{\partial h_1}{\partial x} = A \cdot f \cdot \cos(f \cdot x + \omega \cdot t)$$

#### 3. 折射偏移（Refraction）

基于法线近似值计算折射偏移：

$$\Delta \mathbf{uv} = (n_x, n_y) \times \text{refraction}$$

折射偏移量与法线分量成正比，法线越陡（波纹越剧烈），折射偏移越大。

#### 4. 高光反射（Specular Highlight）

使用 Phong 高光模型计算水面反射光斑：

$$\text{specular} = \left(\max(\text{dot}(\hat{\mathbf{n}}, \hat{\mathbf{v}}), 0)\right)^{32}$$

其中 $\hat{\mathbf{n}} = \text{normalize}(n_x, n_y, 1)$ 为近似法线，$\hat{\mathbf{v}} = (0, 0, 1)$ 为视线方向。指数 32 产生集中的高光光斑。

#### 5. 深度明暗（Depth Shading）

波峰（正值）变亮，波谷（负值）变暗，增强水面的立体感：

$$C_{depth} = (h_1 + h_2 + h_3) \times 10 \times 0.05$$

### 逐段代码分析

#### 段落一：着色器声明与 Uniform 变量

标准声明结构。水波纹效果使用 4 个浮点参数控制波纹的物理特性。

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

#### 段落二：多层正弦波叠加

三个正弦波分别沿不同方向传播，频率和速度各不相同。第一波沿水平方向，第二波沿垂直方向（频率和速度略有不同），第三波沿对角方向（频率最低、振幅最小）。

```glsl
void main() {
    // 多层正弦波叠加模拟水波纹
    vec2 uv = vUV;
    float wave1 = sin(uv.x * uParamFloat1 + uTime * uParamFloat2) * uParamFloat0;
    float wave2 = sin(uv.y * uParamFloat1 * 0.8 + uTime * uParamFloat2 * 1.3) * uParamFloat0 * 0.7;
    float wave3 = sin((uv.x + uv.y) * uParamFloat1 * 0.5 + uTime * uParamFloat2 * 0.7) * uParamFloat0 * 0.5;
```

#### 段落三：法线近似与折射偏移

通过对波高函数求偏导数（正弦的导数为余弦），得到水面法线的近似值。法线分量乘以折射强度参数得到 UV 偏移量，用于采样偏移后的纹理。

```glsl
    // 法线近似 (偏导数)
    float dx = cos(uv.x * uParamFloat1 + uTime * uParamFloat2) * uParamFloat0 * uParamFloat1
             + cos((uv.x + uv.y) * uParamFloat1 * 0.5 + uTime * uParamFloat2 * 0.7) * uParamFloat0 * 0.5 * uParamFloat1 * 0.5;
    float dy = cos(uv.y * uParamFloat1 * 0.8 + uTime * uParamFloat2 * 1.3) * uParamFloat0 * 0.7 * uParamFloat1 * 0.8
             + cos((uv.x + uv.y) * uParamFloat1 * 0.5 + uTime * uParamFloat2 * 0.7) * uParamFloat0 * 0.5 * uParamFloat1 * 0.5;

    // 折射偏移
    vec2 refractOffset = vec2(dx, dy) * uParamFloat3;
    vec3 color = texture(uInputTex, uv + refractOffset).rgb;
```

#### 段落四：高光反射与深度明暗

高光使用 Phong 模型计算法线与视线的点积的 32 次方，产生集中的光斑。深度明暗通过波高总和的缩放实现波峰亮、波谷暗的效果。

```glsl
    // 高光 (模拟水面反射)
    float specular = pow(max(dot(normalize(vec3(dx, dy, 1.0)), normalize(vec3(0.0, 0.0, 1.0))), 0.0), 32.0);
    color += specular * 0.15;

    // 深度感 (波峰亮波谷暗)
    float depth = (wave1 + wave2 + wave3) * 10.0;
    color += depth * 0.05;

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
    float uParamFloat0;  // 振幅
    float uParamFloat1;  // 频率
    float uParamFloat2;  // 速度
    float uParamFloat3;  // 折射强度
    float uParamFloat4;  // 保留参数
    float uParamFloat5;  // 保留参数
    vec2 uResolution;    // 屏幕分辨率
    float uTime;         // 时间
    float uFrameCount;   // 帧计数
};

void main() {
    // === 多层正弦波叠加 ===
    // 三层不同方向、频率和速度的正弦波，模拟复杂的水面干涉图案
    vec2 uv = vUV;
    // 第一波：水平方向，基准频率和速度
    float wave1 = sin(uv.x * uParamFloat1 + uTime * uParamFloat2) * uParamFloat0;
    // 第二波：垂直方向，频率略低(0.8x)，速度略快(1.3x)，振幅0.7x
    float wave2 = sin(uv.y * uParamFloat1 * 0.8 + uTime * uParamFloat2 * 1.3) * uParamFloat0 * 0.7;
    // 第三波：对角方向，频率最低(0.5x)，速度最慢(0.7x)，振幅0.5x
    float wave3 = sin((uv.x + uv.y) * uParamFloat1 * 0.5 + uTime * uParamFloat2 * 0.7) * uParamFloat0 * 0.5;

    // === 法线近似（偏导数）===
    // 对波高函数求x和y方向的偏导数（正弦的导数为余弦）
    // dx: 第一波和第三波的x方向偏导数之和
    float dx = cos(uv.x * uParamFloat1 + uTime * uParamFloat2) * uParamFloat0 * uParamFloat1
             + cos((uv.x + uv.y) * uParamFloat1 * 0.5 + uTime * uParamFloat2 * 0.7) * uParamFloat0 * 0.5 * uParamFloat1 * 0.5;
    // dy: 第二波和第三波的y方向偏导数之和
    float dy = cos(uv.y * uParamFloat1 * 0.8 + uTime * uParamFloat2 * 1.3) * uParamFloat0 * 0.7 * uParamFloat1 * 0.8
             + cos((uv.x + uv.y) * uParamFloat1 * 0.5 + uTime * uParamFloat2 * 0.7) * uParamFloat0 * 0.5 * uParamFloat1 * 0.5;

    // === 折射偏移 ===
    // 法线分量乘以折射强度，得到UV偏移量
    vec2 refractOffset = vec2(dx, dy) * uParamFloat3;
    // 使用偏移后的UV采样输入纹理
    vec3 color = texture(uInputTex, uv + refractOffset).rgb;

    // === 高光反射 ===
    // Phong高光模型：法线与视线方向的点积的32次方
    // 法线近似为(dx, dy, 1)，视线方向为(0, 0, 1)
    float specular = pow(max(dot(normalize(vec3(dx, dy, 1.0)), normalize(vec3(0.0, 0.0, 1.0))), 0.0), 32.0);
    color += specular * 0.15; // 高光强度0.15

    // === 深度明暗 ===
    // 波峰（正值）变亮，波谷（负值）变暗
    float depth = (wave1 + wave2 + wave3) * 10.0;
    color += depth * 0.05; // 深度影响系数0.05

    // 输出最终颜色，限制到[0,1]范围
    outColor = vec4(clamp(color, 0.0, 1.0), 1.0);
}
```

---