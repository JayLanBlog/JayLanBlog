---
title: "CRT 显示器效果 (CRT Display)"
published: 2026-06-11
description: "Shader Showcase 后处理特效系列 — CRT 显示器效果 (CRT Display)详解，包含算法原理、参数说明与完整注释源码"
tags: ["Shader", "GLSL", "后处理", "CRT", "复古", "显示器"]
category: "Shader分析"
draft: false
---

> 本文是 **Shader Showcase 后处理特效着色器源码详解** 系列的第 16 篇。

### 效果概述

CRT 显示器着色器属于 **Retro（复古怀旧）** 类别，模拟了阴极射线管（CRT）显示器的典型视觉特征。该着色器综合了五种 CRT 特征效果：桶形畸变（Barrel Distortion）模拟 CRT 屏幕的曲面玻璃形状；扫描线（Scanlines）模拟电子束逐行扫描的明暗交替；RGB 磷光点遮罩（Phosphor Mask）模拟 CRT 荧光粉的子像素排列；闪烁（Flicker）模拟 CRT 的刷新频率不稳定；暗角（Vignette）模拟 CRT 屏幕边缘的亮度衰减。该效果还包含超出屏幕范围的柔和暗边处理和亮度补偿机制。

### 参数说明

| 参数名 | 标签 | 类型 | 最小值 | 最大值 | 默认值 | UI 类型 | 说明 |
|--------|------|------|--------|--------|--------|---------|------|
| `uParamFloat0` | 扫描线强度 | Float | 0.0 | 1.0 | 0.15 | slider | 扫描线暗化程度 |
| `uParamFloat1` | 屏幕弯曲 | Float | 0.0 | 0.05 | 0.01 | slider | 桶形畸变强度 |
| `uParamFloat2` | RGB遮罩 | Float | 0.0 | 1.0 | 0.3 | slider | 磷光点遮罩的可见程度 |
| `uParamFloat3` | 亮度 | Float | 0.0 | 2.0 | 1.0 | slider | 整体亮度补偿 |
| `uParamFloat4` | 闪烁 | Float | 0.0 | 0.1 | 0.01 | slider | 闪烁强度 |

### 算法原理

#### 1. 桶形畸变（Barrel Distortion）

CRT 屏幕的曲面玻璃使图像产生向外凸起的桶形畸变。该效果通过径向距离的二次函数模拟：

$$\mathbf{uv}_{curved} = \mathbf{uv}_{centered} \times (1 + k \times r^2) + 0.5$$

其中 $k$ 为畸变参数，$r^2 = \|\mathbf{uv}_{centered}\|^2$ 为到屏幕中心的距离平方。正值产生桶形畸变（边缘向外膨胀），负值产生枕形畸变（边缘向内收缩）。

#### 2. 超出范围处理（Border Fade）

当桶形畸变导致 UV 坐标超出 [0, 1] 范围时，不是简单地截断，而是计算超出量并产生柔和的暗边过渡：

$$\text{excess} = |\mathbf{uv}_{curved} - 0.5| - 0.5$$

$$\text{borderFade} = 1 - \text{clamp}(\max(\text{excess}_x, \text{excess}_y) \times 3, 0.15, 0.4)$$

#### 3. 扫描线（Scanlines）

与 VHS 效果类似，但 CRT 的扫描线更微弱：

$$\text{scanline}(y) = \sin(y \times H \times \pi) \times 0.5 + 0.5$$

$$C_{scan} = C \times \max(1 - \text{scanline} \times s \times 0.25, 0.92)$$

注意这里设置了最低亮度 0.92，防止扫描线过于明显。

#### 4. RGB 磷光点遮罩（Phosphor Mask）

CRT 显示器的每个像素由红、绿、蓝三个荧光粉点组成，呈水平排列。遮罩通过像素位置的模 3 运算模拟：

$$m = \text{mod}(\text{pixel}_x, 3)$$

$$\text{mask} = \begin{cases} 1.0 & m < 1 \text{ (红)} \\ 0.93 & 1 \leq m < 2 \text{ (绿)} \\ 0.87 & 2 \leq m < 3 \text{ (蓝)} \end{cases}$$

#### 5. 闪烁与暗角

闪烁通过 60Hz 正弦波模拟 CRT 的刷新频率不稳定：

$$\text{flicker} = 1 - f \times \sin(t \times 60) \times 0.2$$

暗角通过径向距离的二次函数模拟屏幕边缘亮度衰减：

$$\text{vignette} = 1 - r^2 \times 0.35$$

### 逐段代码分析

#### 段落一：着色器声明与 Uniform 变量

标准声明结构。CRT 效果使用了全部 5 个浮点参数和屏幕分辨率。

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

#### 段落二：桶形畸变与超出范围处理

将 UV 坐标以画面中心为原点，计算到中心的距离平方。通过二次函数产生桶形畸变。超出 [0, 1] 范围的部分计算柔和暗边过渡，而非硬截断。

```glsl
void main() {
    vec2 uv = vUV - 0.5; // 以中心为原点
    float dist2 = dot(uv, uv); // 到中心的距离平方

    // 屏幕弯曲 (barrel distortion)
    vec2 curvedUV = uv * (1.0 + uParamFloat1 * dist2) + 0.5;

    // 超出范围: 柔和暗边
    vec2 clampedUV = clamp(curvedUV, 0.0, 1.0);
    float borderFade = 1.0;
    if (curvedUV.x < 0.0 || curvedUV.x > 1.0 ||
        curvedUV.y < 0.0 || curvedUV.y > 1.0) {
        vec2 excess = abs(curvedUV - 0.5) - 0.5;
        borderFade = 1.0 - clamp(max(excess.x, excess.y) * 3.0, 0.15, 0.4);
    }
```

#### 段落三：扫描线与磷光点遮罩

扫描线通过正弦函数产生微弱的明暗交替。磷光点遮罩通过像素水平位置的模 3 运算，对 RGB 三通道施加不同的衰减系数，模拟 CRT 荧光粉的子像素排列。

```glsl
    vec3 color = texture(uInputTex, clampedUV).rgb;

    // 扫描线 — 微弱可见
    float scanline = sin(curvedUV.y * uResolution.y * 3.14159) * 0.5 + 0.5;
    float scanDarken = 1.0 - scanline * uParamFloat0 * 0.25;
    color *= max(scanDarken, 0.92);

    // RGB 磷光点遮罩
    vec2 pixelPos = curvedUV * uResolution;
    float modX = mod(pixelPos.x, 3.0);
    float mask = (modX < 1.0) ? 1.0 : ((modX < 2.0) ? 0.93 : 0.87);
    color *= mix(1.0, mask, uParamFloat2);
```

#### 段落四：闪烁、暗角与亮度补偿

闪烁通过 60Hz 正弦波模拟。暗角通过径向距离衰减边缘亮度。最后应用暗边淡出和亮度补偿（含暗部提亮），确保 CRT 效果不会使画面过暗。

```glsl
    // 闪烁
    float flicker = 1.0 - uParamFloat4 * sin(uTime * 60.0) * 0.2;
    color *= flicker;

    // 边缘暗角 — 轻柔
    float vignette = 1.0 - dist2 * 0.35;
    color *= max(vignette, 0.65);

    color *= borderFade;

    // 亮度补偿 + 暗部提亮
    float brightness = 1.0 + uParamFloat3 * 2.0;
    color = color * brightness + 0.05;  // 暗部也加一点

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
    float uParamFloat1;  // 屏幕弯曲
    float uParamFloat2;  // RGB遮罩
    float uParamFloat3;  // 亮度
    float uParamFloat4;  // 闪烁
    float uParamFloat5;  // 保留参数
    vec2 uResolution;    // 屏幕分辨率
    float uTime;         // 时间
    float uFrameCount;   // 帧计数
};

void main() {
    // 将UV坐标以画面中心为原点
    vec2 uv = vUV - 0.5;
    // 计算到屏幕中心的距离平方
    float dist2 = dot(uv, uv);

    // === 桶形畸变 (Barrel Distortion) ===
    // 通过径向距离的二次函数模拟CRT曲面玻璃的畸变效果
    vec2 curvedUV = uv * (1.0 + uParamFloat1 * dist2) + 0.5;

    // === 超出范围处理 ===
    // 畸变后UV超出[0,1]时，产生柔和暗边过渡而非硬截断
    vec2 clampedUV = clamp(curvedUV, 0.0, 1.0);
    float borderFade = 1.0;
    if (curvedUV.x < 0.0 || curvedUV.x > 1.0 ||
        curvedUV.y < 0.0 || curvedUV.y > 1.0) {
        vec2 excess = abs(curvedUV - 0.5) - 0.5; // 计算超出量
        borderFade = 1.0 - clamp(max(excess.x, excess.y) * 3.0, 0.15, 0.4);
    }

    // 使用畸变后的UV采样输入纹理
    vec3 color = texture(uInputTex, clampedUV).rgb;

    // === 扫描线 ===
    // 通过正弦函数产生微弱的明暗交替，模拟CRT电子束逐行扫描
    float scanline = sin(curvedUV.y * uResolution.y * 3.14159) * 0.5 + 0.5;
    float scanDarken = 1.0 - scanline * uParamFloat0 * 0.25;
    color *= max(scanDarken, 0.92); // 最低亮度0.92，防止扫描线过强

    // === RGB磷光点遮罩 ===
    // 通过像素水平位置的模3运算，模拟CRT荧光粉子像素排列
    vec2 pixelPos = curvedUV * uResolution;
    float modX = mod(pixelPos.x, 3.0);
    float mask = (modX < 1.0) ? 1.0 : ((modX < 2.0) ? 0.93 : 0.87);
    color *= mix(1.0, mask, uParamFloat2); // 通过mix控制遮罩可见度

    // === 闪烁 ===
    // 通过60Hz正弦波模拟CRT刷新频率不稳定
    float flicker = 1.0 - uParamFloat4 * sin(uTime * 60.0) * 0.2;
    color *= flicker;

    // === 暗角 ===
    // 通过径向距离衰减边缘亮度，模拟CRT屏幕边缘变暗
    float vignette = 1.0 - dist2 * 0.35;
    color *= max(vignette, 0.65); // 最低亮度0.65，防止暗角过重

    // 应用暗边淡出
    color *= borderFade;

    // === 亮度补偿 + 暗部提亮 ===
    float brightness = 1.0 + uParamFloat3 * 2.0;
    color = color * brightness + 0.05; // 加0.05确保暗部也有一定亮度

    // 输出最终颜色，限制到[0,1]范围
    outColor = vec4(clamp(color, 0.0, 1.0), 1.0);
}
```

---