---
title: "调色效果 (Color Grading)"
published: 2026-06-11
description: "Shader Showcase 后处理特效系列 — 调色效果 (Color Grading)详解，包含算法原理、参数说明与完整注释源码"
tags: ["Shader", "GLSL", "后处理", "调色", "色彩"]
category: "Shader分析"
draft: false
---

> 本文是 **Shader Showcase 后处理特效着色器源码详解** 系列的第 10 篇。

### 效果概述

调色着色器（Color Grading）属于 **Color（色彩校正）** 类别，是电影级后期处理中最核心的效果之一。该着色器模拟了专业色彩分级工作流中的五个关键参数：曝光（Exposure）、对比度（Contrast）、饱和度（Saturation）、色温（Color Temperature）和色调偏移（Tint）。通过这些参数的组合调整，用户可以快速为画面赋予不同的视觉风格——从冷峻的蓝调电影感，到温暖的复古胶片感，再到高对比度的戏剧化效果。色温转换采用了 Tanner Helland 提出的简化算法，将开尔文温度值映射为 RGB 颜色，并以 6500K（D65 标准白光）作为中性参考点进行相对偏移计算。

### 参数说明

| 参数名 | 标签 | 类型 | 最小值 | 最大值 | 默认值 | UI 类型 | 说明 |
|--------|------|------|--------|--------|--------|---------|------|
| `uParamFloat0` | 曝光 | Float | -3.0 | 3.0 | 0.0 | slider | 控制画面整体亮度，基于 2 的幂次方调整 |
| `uParamFloat1` | 对比度 | Float | -1.0 | 1.0 | 0.0 | slider | 以 0.5 为中心点的对比度拉伸/压缩 |
| `uParamFloat2` | 饱和度 | Float | -1.0 | 1.0 | 0.0 | slider | 颜色饱和度调整，-1 为完全灰度，+1 为过度饱和 |
| `uParamFloat3` | 色温 | Float | 1000.0 | 40000.0 | 6500.0 | slider | 开尔文色温值，6500K 为标准白光 |
| `uParamFloat4` | 色调 | Float | -1.0 | 1.0 | 0.0 | slider | 绿-品红轴色调偏移 |

### 算法原理

#### 1. 曝光调整（Exposure）

曝光调整基于摄影学中的曝光值（EV）概念，采用 2 的幂次方函数：

$$C_{exposure} = C_{original} \times 2^{EV}$$

其中 $EV$ 为曝光参数值。当 $EV = 0$ 时画面不变；$EV > 0$ 时画面变亮（每增加 1 档亮度翻倍）；$EV < 0$ 时画面变暗。这种对数线性映射与人眼的亮度感知更吻合。

#### 2. 对比度调整（Contrast）

对比度以 0.5（中灰）为中心点进行线性拉伸：

$$C_{contrast} = (C_{exposure} - 0.5) \times (1 + k) + 0.5$$

其中 $k$ 为对比度参数。当 $k > 0$ 时，亮部更亮、暗部更暗，画面反差增大；当 $k < 0$ 时，画面趋向灰蒙蒙的平淡效果。

#### 3. 饱和度调整（Saturation）

饱和度调整通过在原始颜色与灰度亮度之间进行线性插值实现：

$$L = 0.2126 \cdot R + 0.7152 \cdot G + 0.0722 \cdot B$$

$$C_{sat} = \text{mix}(L, C_{contrast}, 1 + s)$$

其中 $L$ 是基于 ITU-R BT.709 标准的亮度权重，$s$ 为饱和度参数。当 $s = -1$ 时，权重变为 0，输出完全灰度图像；当 $s = 0$ 时保持原色。

#### 4. 色温转换（Color Temperature）

色温转换采用 Tanner Helland 的近似算法，将开尔文温度 $T$ 映射为 RGB 三通道值。核心思路是分段定义各通道在不同温度区间的响应曲线：

- **红色通道**：$T \leq 6600K$ 时为 1.0（全红）；$T > 6600K$ 时按幂律衰减
- **绿色通道**：在 $T \leq 6600K$ 时为对数曲线，$T > 6600K$ 时为幂律曲线
- **蓝色通道**：$T \leq 1900K$ 时为 0（无蓝）；$1900K < T < 6600K$ 时为对数曲线；$T \geq 6600K$ 时为 1.0

最终以 6500K（D65 标准白光）为中性参考点，计算相对偏移：

$$C_{temp} = C_{sat} \times \frac{RGB(T)}{RGB(6500)}$$

#### 5. 色调偏移（Tint）

色调偏移沿绿-品红轴进行简单线性偏移：

$$C_{final} = C_{temp}, \quad C_{final}.g += t \times 0.1$$

其中 $t$ 为色调参数，正值偏绿，负值偏品红。

### 逐段代码分析

#### 段落一：着色器声明与 Uniform 变量

该段定义了 GLSL 460 版本的着色器输入输出和统一变量块。`vUV` 为顶点着色器传递的纹理坐标，`uInputTex` 为输入纹理采样器，`Params` 统一块包含了 6 个浮点参数（其中前 5 个用于调色控制）、屏幕分辨率、时间和帧计数。

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

#### 段落二：开尔文色温转 RGB 函数

`kelvinToRGB` 函数实现了 Tanner Helland 的色温近似算法。输入为开尔文温度值，输出为归一化的 RGB 颜色。函数内部按温度区间对 R、G、B 三通道分别计算，使用 `pow`（幂函数）和 `log`（对数函数）来模拟黑体辐射的光谱分布特征。所有结果通过 `clamp` 限制在 [0, 1] 范围内。

```glsl
vec3 kelvinToRGB(float temp) {
    float t = temp / 100.0;
    vec3 color;
    // Red
    if (t <= 66.0) color.r = 1.0;
    else color.r = clamp(1.292936 * pow(t - 60.0, -0.133204), 0.0, 1.0);
    // Green
    if (t <= 66.0) color.g = clamp(0.39008 * log(t) - 0.63184, 0.0, 1.0);
    else color.g = clamp(1.12989 * pow(t - 60.0, -0.07551), 0.0, 1.0);
    // Blue
    if (t >= 66.0) color.b = 1.0;
    else if (t <= 19.0) color.b = 0.0;
    else color.b = clamp(0.54320 * log(t - 10.0) - 0.27556, 0.0, 1.0);
    return color;
}
```

#### 段落三：主函数 — 曝光、对比度、饱和度处理

主函数首先采样原始纹理颜色，然后依次进行曝光（`exp2` 对数调整）、对比度（以 0.5 为中心的线性拉伸）、饱和度（基于 BT.709 亮度权重的 mix 插值）三项处理。曝光使用 `exp2` 而非简单的乘法，是因为摄影中的曝光值本身就是以 2 为底的对数单位。

```glsl
void main() {
    vec3 color = texture(uInputTex, vUV).rgb;

    // Exposure
    color *= exp2(uParamFloat0);

    // Contrast (around 0.18 midpoint)
    color = (color - 0.5) * (1.0 + uParamFloat1) + 0.5;

    // Saturation
    float luma = dot(color, vec3(0.2126, 0.7152, 0.0722));
    color = mix(vec3(luma), color, 1.0 + uParamFloat2);
```

#### 段落四：主函数 — 色温偏移与色调调整

色温处理通过分别计算目标温度和 6500K 中性白光的 RGB 值，然后取比值得到偏移因子，乘以当前颜色实现色温偏移。色调偏移则直接在绿色通道上叠加一个线性偏移量，模拟绿-品红轴的色彩偏移。最终结果通过 `clamp` 限制到 [0, 1] 范围。

```glsl
    // Temperature
    vec3 kelvinColor = kelvinToRGB(uParamFloat3);
    vec3 kelvinNeutral = kelvinToRGB(6500.0);
    vec3 tempTint = kelvinColor / kelvinNeutral;
    color *= tempTint;

    // Tint (green-magenta shift)
    color.g += uParamFloat4 * 0.1;

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
// 统一参数块（std140 布局）
layout(std140, binding=1) uniform Params {
    float uParamFloat0;  // 曝光
    float uParamFloat1;  // 对比度
    float uParamFloat2;  // 饱和度
    float uParamFloat3;  // 色温
    float uParamFloat4;  // 色调
    float uParamFloat5;  // 保留参数
    vec2 uResolution;    // 屏幕分辨率
    float uTime;         // 时间
    float uFrameCount;   // 帧计数
};

// 简化的色温转RGB (Tanner Helland算法)
// 将开尔文温度值转换为归一化的RGB颜色
vec3 kelvinToRGB(float temp) {
    float t = temp / 100.0; // 将温度缩放到合理范围
    vec3 color;
    // 红色通道：低温时为1.0，高温时按幂律衰减
    if (t <= 66.0) color.r = 1.0;
    else color.r = clamp(1.292936 * pow(t - 60.0, -0.133204), 0.0, 1.0);
    // 绿色通道：低温时为对数曲线，高温时为幂律曲线
    if (t <= 66.0) color.g = clamp(0.39008 * log(t) - 0.63184, 0.0, 1.0);
    else color.g = clamp(1.12989 * pow(t - 60.0, -0.07551), 0.0, 1.0);
    // 蓝色通道：极低温时为0，高温时为1.0，中间为对数曲线
    if (t >= 66.0) color.b = 1.0;
    else if (t <= 19.0) color.b = 0.0;
    else color.b = clamp(0.54320 * log(t - 10.0) - 0.27556, 0.0, 1.0);
    return color;
}

void main() {
    // 采样输入纹理的原始颜色
    vec3 color = texture(uInputTex, vUV).rgb;

    // 曝光调整：使用2的幂次方函数，模拟摄影曝光值(EV)
    color *= exp2(uParamFloat0);

    // 对比度调整：以0.5为中心点进行线性拉伸
    color = (color - 0.5) * (1.0 + uParamFloat1) + 0.5;

    // 饱和度调整：基于ITU-R BT.709亮度权重计算灰度值
    // 在灰度和原色之间插值，控制饱和度
    float luma = dot(color, vec3(0.2126, 0.7152, 0.0722));
    color = mix(vec3(luma), color, 1.0 + uParamFloat2);

    // 色温调整：计算目标温度的RGB，以6500K(D65标准白光)为中性参考
    vec3 kelvinColor = kelvinToRGB(uParamFloat3);
    vec3 kelvinNeutral = kelvinToRGB(6500.0);
    vec3 tempTint = kelvinColor / kelvinNeutral; // 相对偏移因子
    color *= tempTint;

    // 色调偏移：沿绿-品红轴进行线性偏移
    color.g += uParamFloat4 * 0.1;

    // 输出最终颜色，限制到[0,1]范围
    outColor = vec4(clamp(color, 0.0, 1.0), 1.0);
}
```

---