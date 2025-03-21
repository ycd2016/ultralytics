# YOLO 系列提升

[下载权重](https://drive.google.com/drive/folders/1jaH9J0cQ-5ypgHovUtRlylz4FOX4hhOr?usp=sharing)

## 项目简介

我询问Claude 3.7要如何在保持推理速度基本不变的情况下，通过结构创新和损失函数优化实现YOLO系列的精度提升，最终生成了本项目的代码。
保持其它超参数不变，在自己的机器上重新训练了所有模型，并发现真的有用，现将其代码及模型开源。以下改进内容由Deepseek R1根据代码生成。

## 优化后性能

我没有 T4 可供测速，所以删除了速度对比那一列。

| 原始模型 | COCO mAP 640px | mAP Comparison | Params (M) | FLOPs (G) | COCO mAP 960px |
|----------|----------------|----------------|------------|-----------|----------------|
| 12n      | 41.6           | +1.2           | 2.6        | 6.5       | 46.5           |
| 12s      | 49.3           | +1.7           | 9.3        | 21.5      | 52.8           |
| 12m      | 54.4           | +1.9           | 20.3       | 67.6      | 55.6           |
| 12l      | 55.5           | +1.7           | 26.5       | 89.1      | 57.5           |
| 12x      | 57.6           | +2.2           | 59.4       | 199.4     | 58.7           |

<figure>
    <img src="flops.png" alt="flops" style="width: 50%;"><img src="params.png" alt="params" style="width: 50%;">
</figure>

## 一、异构卷积

### 核心思想

在原始卷积层的基础上，增加了一个并行的 1×1 卷积操作，并在前向传播中将两个卷积的输出相加。

### 改进后的模块

通过并行双路径结构实现了：

1. **多尺度特征融合**：3×3卷积捕获局部空间特征，1×1卷积提取全局上下文信息，通过线性叠加实现特征增强：

```math
Z = \mathcal{F}_{3×3}(X) + \mathcal{F}_{1×1}(X)
```

2. **参数效率优化**：1×1卷积作为轻量级补充分支，仅增加 $C^2$ 参数，实现计算量 $\mathcal{O}(C^2)$ 级的优化。

### 理论优势

1. **特征表达增强**：双分支结构扩展了假设空间，使网络能学习更复杂的特征组合。根据通用近似定理，双线性组合使模型容量提升：

```math
\exists W_1,W_2 \quad s.t. \quad ||f(x)-(W_1^T\phi_1(x)+W_2^T\phi_2(x))|| < \epsilon
```

2. **梯度传播优化**：反向传播时梯度流分为两路，缓解梯度消失问题：

```math
\frac{\partial \mathcal{L}}{\partial X} = \frac{\partial \mathcal{L}}{\partial Z} \left( \frac{\partial \mathcal{F}_{3×3}}{\partial X} + \frac{\partial \mathcal{F}_{1×1}}{\partial X} \right)
```

3. **推理加速机制**：通过卷积核融合技术，将双卷积合并为等效单卷积：

```math
W_{fused} = W_{3×3} + \hat{W}_{1×1}
```

其中 $\hat{W}_{1×1}$ 是将 1×1 卷积核 zero-padding 为 3×3 形式的扩展表示。

## 二、EIoU

### 核心改进

引入了 Elastic IoU，这是一个更为鲁棒的 IoU 计算方法，特别适用于小而密集的目标。

```math
\mathcal{L}_{EIoU} = 1 - IoU + \frac{\rho^2(b,b^{gt})}{c^2} + \beta \cdot \tanh(\sqrt{A \cdot A^{gt}}/\tau)\cdot v
```

其中：

- $\beta=1.5$ 控制惩罚项的强度
- $\tau=75$ 调节尺寸敏感度的温度系数
- $v$ 为长宽比一致性项

### 优势分析

1. **动态惩罚机制**：
- 原版 CIoU 使用固定权重组合中心距离与长宽比项：

```math
\mathcal{L}_{CIoU} = 1 - IoU + \frac{\rho^2}{c^2} + \beta v
```

- 改进版 EIoU 引入自适应权重：

```math
\mathcal{L}_{EIoU} = 1 - IoU + \underbrace{\omega (\frac{\rho^2}{c^2} + 0.3v)}_{\text{弹性惩罚项}}
```

  其中权重项 $\omega = (1-IoU)^\beta(1+\alpha)\in[0.5,3]$，β=1.5 为稳定超参数。

2. **小目标优化**：
新增面积敏感函数：

```math
\alpha = 1 - \tanh\left(\frac{\sqrt{A_1A_2}}{75}\right)
```

该函数在 $A<75^2$ 时输出接近 1，使小目标获得更强的梯度回传。相比原版 CIoU，对小目标的定位误差敏感度显著提升：

```math
\frac{\partial \mathcal{L}_{EIoU}}{\partial \rho} \propto \frac{2\rho}{c^2}(1+\alpha) > \frac{2\rho}{c^2}
```

3. **数值稳定性增强**：
- 引入尺寸下限约束：

```math
w_{safe} = \max(w, \epsilon)
```

- 使用 arctan 替代原始宽高比计算，将比值差异映射到 $(-π/2, π/2)$ 区间，防止梯度爆炸：

```math
\Delta\theta = \arctan\frac{w_2}{h_2} - \arctan\frac{w_1}{h_1} \in (-π, π)
```

## 三、协同注意力

### 结构设计

引入了轻量级通道注意力模块和简化的空间注意力模块。

### 轻量级通道注意力

引入的通道注意力模块采用双重压缩-激励策略，其数学表达为：

```math
\begin{aligned}
&z_c = \mathcal{F}_{GAP}(X) \\
&w_c = \sigma(W_2 \cdot \delta(W_1 \cdot z_c)) \\
&Y_c = X \odot (1 + \alpha w_c) \quad (\alpha=0.5)
\end{aligned}
```

其中 $\mathcal{F}_{GAP}$ 为全局平均池化，$\delta$ 为 SiLU 激活函数，$\sigma$ 为 Sigmoid 函数。相比传统 SE 模块，改进在于：

1. 通道压缩率提升至 1/8，保留更多有效信息。
2. 采用动态权重缩放机制，避免特征过度调制。

### 空间注意力优化

空间注意力模块采用轻量化设计：

```math
w_s = \sigma(BN(Conv_{1\times1}(X)))
```

该设计特点：

1. 单层 3×3 卷积替代传统双层结构。
2. BatchNorm 层稳定梯度分布，改善小目标检测。

### 数学表达

```math
X_{enhanced} = X \odot (1+\frac{1}{2}SE(X)) \odot (1+\frac{1}{2}SA(X))
```

### 优势分析

1. **特征校准**：通道注意力通过压缩-激励机制增强重要特征通道。
2. **空间聚焦**：空间注意力使用轻量级卷积定位关键区域。
3. **梯度稳定**：采用 0.5 的缩放因子避免特征图过饱和。
