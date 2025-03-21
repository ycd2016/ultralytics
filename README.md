# YOLO Series Improvement

[Download weights](https://drive.google.com/drive/folders/1jaH9J0cQ-5ypgHovUtRlylz4FOX4hhOr?usp=sharing)

## Project Overview

I asked Claude 3.7 how to improve the accuracy of the YOLO series through structural innovation and loss function optimization while keeping the inference speed essentially unchanged. Based on the code generated by Claude, I retrained all models on my machine with the same hyperparameters and found that the improvements were indeed effective. Here is the code and models open-sourced. The following improvements were generated by Deepseek R1 based on the code.

## Optimized Performance

I don't have a T4 for speed testing, so I removed the speed comparison column.

| Original Model | COCO mAP 640px | mAP Comparison | Params (M) | FLOPs (G) | COCO mAP 960px |
|----------------|----------------|----------------|------------|-----------|----------------|
| 12n            | 41.6           | +1.2           | 2.6        | 6.5       | 46.5           |
| 12s            | 49.3           | +1.7           | 9.3        | 21.5      | 52.8           |
| 12m            | 54.4           | +1.9           | 20.3       | 67.6      | 55.6           |
| 12l            | 55.5           | +1.7           | 26.5       | 89.1      | 57.5           |
| 12x            | 57.6           | +2.2           | 59.4       | 199.4     | 58.7           |

<figure>
    <img src="flops.png" alt="flops" style="width: 50%;"><img src="params.png" alt="params" style="width: 50%;">
</figure>

## I. Heterogeneous Convolution

### Core Idea

In addition to the original convolution layer, a parallel 1×1 convolution operation is added, and the outputs of the two convolutions are summed during forward propagation.

### Improved Module

Implemented through a parallel dual-path structure:

1. **Multi-scale Feature Fusion**: The 3×3 convolution captures local spatial features, while the 1×1 convolution extracts global contextual information, achieving feature enhancement through linear superposition:

```math
Z = \mathcal{F}_{3×3}(X) + \mathcal{F}_{1×1}(X)
```

2. **Parameter Efficiency Optimization**: The 1×1 convolution serves as a lightweight complementary branch, adding only $C^2$ parameters, achieving computational optimization at the $\mathcal{O}(C^2)$ level.

### Theoretical Advantages

1. **Enhanced Feature Representation**: The dual-branch structure expands the hypothesis space, allowing the network to learn more complex feature combinations. According to the universal approximation theorem, the bilinear combination increases model capacity:

```math
\exists W_1,W_2 \quad s.t. \quad ||f(x)-(W_1^T\phi_1(x)+W_2^T\phi_2(x))|| < \epsilon
```

2. **Gradient Propagation Optimization**: During backpropagation, the gradient flow is divided into two paths, alleviating the gradient vanishing problem:

```math
\frac{\partial \mathcal{L}}{\partial X} = \frac{\partial \mathcal{L}}{\partial Z} \left( \frac{\partial \mathcal{F}_{3×3}}{\partial X} + \frac{\partial \mathcal{F}_{1×1}}{\partial X} \right)
```

3. **Inference Acceleration Mechanism**: Through convolution kernel fusion technology, the dual convolutions are merged into an equivalent single convolution:

```math
W_{fused} = W_{3×3} + \hat{W}_{1×1}
```

where $\hat{W}_{1×1}$ is the zero-padded 1×1 convolution kernel expanded to a 3×3 form.

## II. EIoU

### Core Improvement

Introducing Elastic IoU, a more robust IoU calculation method, particularly suitable for small and dense objects.

```math
\mathcal{L}_{EIoU} = 1 - IoU + \frac{\rho^2(b,b^{gt})}{c^2} + \beta \cdot \tanh(\sqrt{A \cdot A^{gt}}/\tau)\cdot v
```

where:

- $\beta=1.5$ controls the strength of the penalty term
- $\tau=75$ adjusts the temperature coefficient for size sensitivity
- $v$ is the aspect ratio consistency term

### Advantage Analysis

1. **Dynamic Penalty Mechanism**:
- The original CIoU uses fixed weights to combine center distance and aspect ratio terms:

```math
\mathcal{L}_{CIoU} = 1 - IoU + \frac{\rho^2}{c^2} + \beta v
```

- The improved EIoU introduces adaptive weights:

```math
\mathcal{L}_{EIoU} = 1 - IoU + \underbrace{\omega (\frac{\rho^2}{c^2} + 0.3v)}_{\text{Elastic Penalty Term}}
```
where the weight term $\omega = (1-IoU)^\beta(1+\alpha)\in[0.5,3]$, β=1.5 is a stable hyperparameter.

2. **Small Object Optimization**:
Added area-sensitive function:

```math
\alpha = 1 - \tanh\left(\frac{\sqrt{A_1A_2}}{75}\right)
```

This function outputs close to 1 when $A<75^2$, giving small objects stronger gradient backpropagation. Compared to the original CIoU, the sensitivity to localization errors for small objects is significantly enhanced:

```math
\frac{\partial \mathcal{L}_{EIoU}}{\partial \rho} \propto \frac{2\rho}{c^2}(1+\alpha) > \frac{2\rho}{c^2}
```

3. **Enhanced Numerical Stability**:
- Introduced size lower bound constraint:

```math
w_{safe} = \max(w, \epsilon)
```

- Used arctan to replace the original aspect ratio calculation, mapping the ratio difference to the $(-π/2, π/2)$ interval, preventing gradient explosion:

```math
\Delta\theta = \arctan\frac{w_2}{h_2} - \arctan\frac{w_1}{h_1} \in (-π, π)
```

## III. Collaborative Attention

### Structural Design

Introduced lightweight channel attention module and simplified spatial attention module.

### Lightweight Channel Attention

The introduced channel attention module adopts a dual compression-excitation strategy, mathematically expressed as:

```math
\begin{aligned}
&z_c = \mathcal{F}_{GAP}(X) \\
&w_c = \sigma(W_2 \cdot \delta(W_1 \cdot z_c)) \\
&Y_c = X \odot (1 + \alpha w_c) \quad (\alpha=0.5)
\end{aligned}
```

where $\mathcal{F}_{GAP}$ is global average pooling, $\delta$ is the SiLU activation function, and $\sigma$ is the Sigmoid function. Compared to the traditional SE module, the improvements are:

1. Channel compression rate increased to 1/8, retaining more effective information.
2. Adopted dynamic weight scaling mechanism to avoid excessive feature modulation.

### Spatial Attention Optimization

The spatial attention module adopts a lightweight design:

```math
w_s = \sigma(BN(Conv_{1\times1}(X)))
```

Design features:

1. Single-layer 3×3 convolution replaces the traditional dual-layer structure.
2. BatchNorm layer stabilizes gradient distribution, improving small object detection.

### Mathematical Expression

```math
X_{enhanced} = X \odot (1+\frac{1}{2}SE(X)) \odot (1+\frac{1}{2}SA(X))
```

### Advantage Analysis

1. **Feature Calibration**: Channel attention enhances important feature channels through compression-excitation mechanism.
2. **Spatial Focus**: Spatial attention uses lightweight convolution to locate key areas.
3. **Gradient Stability**: Using a 0.5 scaling factor to avoid feature map oversaturation.
