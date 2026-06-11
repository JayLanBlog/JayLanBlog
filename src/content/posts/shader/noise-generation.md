---
title: "噪声生成 (Noise Generation)"
published: 2026-06-11
description: "Shader Showcase 后处理特效系列 — 噪声生成 (Noise Generation)详解，包含算法原理、参数说明与完整注释源码"
tags: ["Shader", "GLSL", "后处理", "噪声", "程序化"]
category: "Shader分析"
draft: false
---

> 本文是 **Shader Showcase 后处理特效着色器源码详解** 系列的第 11 篇。

### 效果概述

噪声生成着色器属于 **Procedural（程序化生成）** 类别，实现了基于分形布朗运动（FBM, Fractal Brownian Motion）的 Perlin 噪声叠加效果。该着色器通过多层（5 层）不同频率和振幅的噪声叠加，生成具有自然有机感的纹理图案，可模拟烟雾、云层、水波、大理石纹理等自然现象。噪声图案会随时间动态演变，产生流动的视觉效果。用户可控制噪声强度、纹理缩放比例和动画速度三个参数。

### 参数说明

| 参数名 | 标签 | 类型 | 最小值 | 最大值 | 默认值 | UI 类型 | 说明 |
|--------|------|------|--------|--------|--------|---------|------|
| `uParamFloat0` | 噪声强度 | Float | 0.0 | 1.0 | 0.3 | slider | 噪声叠加到原图上的强度 |
| `uParamFloat1` | 缩放 | Float | 1.0 | 100.0 | 10.0 | slider | 噪声纹理的频率缩放（值越大纹理越细密） |
| `uParamFloat2` | 动画速度 | Float | 0.0 | 5.0 | 1.0 | slider | 噪声随时间流动的速度 |

### 算法原理

#### 1. 哈希函数（Hash Function）

哈希函数将二维坐标映射为伪随机值，是噪声生成的基础。该实现使用了一种基于分形运算的哈希方法：

$$h(\mathbf{p}) = \text{fract}\left(\text{fract}(\mathbf{p} \times 0.1031) + \text{dot}(\mathbf{p}', \mathbf{p}'_{yzx} + 33.33)\right)$$

其中 $\mathbf{p}' = \text{fract}(\mathbf{p}_{xyx} \times 0.1031)$，最终取小数部分得到 [0, 1) 范围的伪随机值。

#### 2. Value Noise（值噪声）

值噪声通过在网格顶点上放置伪随机值，然后在网格内部进行双线性插值生成连续的噪声场：

$$n(\mathbf{p}) = \text{bilinear}\left(h(\lfloor \mathbf{p} \rfloor), h(\lfloor \mathbf{p} \rfloor + (1,0)), h(\lfloor \mathbf{p} \rfloor + (0,1)), h(\lfloor \mathbf{p} \rfloor + (1,1)), \text{fract}(\mathbf{p})\right)$$

插值权重使用 Hermite 平滑步进函数（smoothstep）：

$$f(t) = t^2(3 - 2t)$$

该函数在网格边界处的一阶导数为零，确保噪声场的连续性。

#### 3. 分形布朗运动（FBM）

FBM 通过叠加多个不同频率（octave）的噪声来生成具有多尺度细节的复杂纹理：

$$\text{FBM}(\mathbf{p}) = \sum_{i=0}^{N-1} a_i \cdot n(2^i \cdot \mathbf{p}), \quad a_i = 0.5^i$$

其中 $N = 5$ 为叠加层数，每层频率翻倍（$2^i$），振幅减半（$0.5^i$）。这种 1/f 噪声特性使得生成的图案在不同尺度上都具有自然的细节变化。

#### 4. 噪声叠加到图像

最终将 FBM 噪声值（范围约 [0, 1]）减去 0.5 映射到 [-0.5, 0.5]，再乘以强度参数叠加到原图颜色上：

$$C_{final} = C_{original} + (\text{FBM}(\mathbf{p}) - 0.5) \times \text{strength}$$

### 逐段代码分析

#### 段落一：着色器声明与 Uniform 变量

与调色着色器相同的标准声明结构。包含输入纹理坐标、输出颜色、纹理采样器和参数块。

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

#### 段落二：哈希函数与值噪声

`hash` 函数基于分形运算生成伪随机数，输入二维坐标，输出 [0, 1) 范围的随机值。`noise` 函数实现值噪声：先获取网格四个顶点的哈希值，然后使用 smoothstep 插值权重进行双线性插值，生成连续平滑的噪声场。

```glsl
// Hash functions for noise
float hash(vec2 p) {
    vec3 p3 = fract(vec3(p.xyx) * 0.1031);
    p3 += dot(p3, p3.yzx + 33.33);
    return fract((p3.x + p3.y) * p3.z);
}

float noise(vec2 p) {
    vec2 i = floor(p);       // 网格整数坐标
    vec2 f = fract(p);       // 网格内小数坐标
    f = f * f * (3.0 - 2.0 * f); // smoothstep 插值权重

    float a = hash(i);                // 左下角
    float b = hash(i + vec2(1.0, 0.0)); // 右下角
    float c = hash(i + vec2(0.0, 1.0)); // 左上角
    float d = hash(i + vec2(1.0, 1.0)); // 右上角

    return mix(mix(a, b, f.x), mix(c, d, f.x), f.y); // 双线性插值
}
```

#### 段落三：分形布朗运动（FBM）

FBM 函数通过 5 次循环叠加噪声，每次将坐标频率翻倍（`p *= 2.0`），振幅减半（`amplitude *= 0.5`）。这种多尺度叠加方式产生了从大尺度结构到小尺度细节的丰富纹理。

```glsl
float fbm(vec2 p) {
    float value = 0.0;
    float amplitude = 0.5;
    for (int i = 0; i < 5; i++) {
        value += amplitude * noise(p); // 叠加当前层噪声
        p *= 2.0;                      // 频率翻倍
        amplitude *= 0.5;              // 振幅减半
    }
    return value;
}
```

#### 段落四：主函数 — 噪声采样与叠加

主函数将纹理坐标乘以缩放参数并加上时间偏移（实现动画），然后计算 FBM 噪声值，将其映射到 [-0.5, 0.5] 后乘以强度叠加到原图颜色上。

```glsl
void main() {
    vec3 color = texture(uInputTex, vUV).rgb;
    vec2 uv = vUV * uParamFloat1 + uTime * uParamFloat2; // 缩放+时间偏移
    float n = fbm(uv);                                   // 计算FBM噪声
    color += (n - 0.5) * uParamFloat0;                    // 叠加噪声
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
    float uParamFloat0;  // 噪声强度
    float uParamFloat1;  // 缩放
    float uParamFloat2;  // 动画速度
    float uParamFloat3;  // 保留参数
    float uParamFloat4;  // 保留参数
    float uParamFloat5;  // 保留参数
    vec2 uResolution;    // 屏幕分辨率
    float uTime;         // 时间
    float uFrameCount;   // 帧计数
};

// 哈希函数：将二维坐标映射为伪随机值
// 使用分形运算产生良好的伪随机分布
float hash(vec2 p) {
    vec3 p3 = fract(vec3(p.xyx) * 0.1031); // 坐标缩放并取小数部分
    p3 += dot(p3, p3.yzx + 33.33);          // 混合各分量增加随机性
    return fract((p3.x + p3.y) * p3.z);     // 最终取小数部分
}

// 值噪声函数：在网格顶点间进行平滑插值
float noise(vec2 p) {
    vec2 i = floor(p);       // 获取网格整数坐标（左下角顶点）
    vec2 f = fract(p);       // 获取网格内小数坐标（插值权重）
    f = f * f * (3.0 - 2.0 * f); // Hermite smoothstep 平滑插值

    // 获取网格四个顶点的伪随机值
    float a = hash(i);                // 左下角
    float b = hash(i + vec2(1.0, 0.0)); // 右下角
    float c = hash(i + vec2(0.0, 1.0)); // 左上角
    float d = hash(i + vec2(1.0, 1.0)); // 右上角

    // 双线性插值：先水平插值，再垂直插值
    return mix(mix(a, b, f.x), mix(c, d, f.x), f.y);
}

// 分形布朗运动（FBM）：叠加多层不同频率的噪声
float fbm(vec2 p) {
    float value = 0.0;      // 累积噪声值
    float amplitude = 0.5;  // 初始振幅
    for (int i = 0; i < 5; i++) { // 5层叠加
        value += amplitude * noise(p); // 叠加当前层噪声
        p *= 2.0;                      // 频率翻倍（更细密的纹理）
        amplitude *= 0.5;              // 振幅减半（更小的影响）
    }
    return value;
}

void main() {
    // 采样输入纹理的原始颜色
    vec3 color = texture(uInputTex, vUV).rgb;
    // 将纹理坐标缩放并加上时间偏移，实现动态噪声
    vec2 uv = vUV * uParamFloat1 + uTime * uParamFloat2;
    // 计算FBM噪声值
    float n = fbm(uv);
    // 将噪声从[0,1]映射到[-0.5,0.5]，乘以强度后叠加到原图
    color += (n - 0.5) * uParamFloat0;
    // 输出最终颜色，限制到[0,1]范围
    outColor = vec4(clamp(color, 0.0, 1.0), 1.0);
}
```

---