---
layout: default
title: Batch / Layer / RMS / QK 归一化
date: 2026-05-11 16:08:00
---

## 对比

| 归一化 | 作用维度 | 可学习参数 | 依赖 batch | 典型用途 |
|--------|---------|----------|-----------|---------|
| BatchNorm | B (batch) | $\gamma, \beta$ | 是 | CNN, 视觉任务 |
| LayerNorm | D (feature) | $\gamma, \beta$ | 否 | Transformer, RNN |
| RMSNorm | D (feature) | $\gamma$ | 否 | LLaMA, 大语言模型 |
| QK Norm | d_k (per head) | $\alpha$ | 否 | 大模型注意力稳定 |

**选择建议**：现代 LLM 架构普遍采用 **RMSNorm + QK Norm** 的组合，在训练稳定性和推理速度之间取得了良好平衡。

---

## 为什么需要归一化

深度网络训练中，层间输出的分布会随参数更新不断漂移（Internal Covariate Shift），导致：

- 梯度消失或爆炸
- 需要更小的学习率
- 对初始化敏感

归一化通过标准化每层输入来缓解这些问题。以下输入统一表示为 $(B, N, D)$，即批量大小、序列长度、特征维度。

---

## Batch Normalization

对 batch 维度做归一化。给定输入 $x \in \mathbb{R}^{B \times N \times D}$，在 B 和 N 维度求均值和方差（即每个特征维度 $D$ 上，跨 batch 和序列位置做统计）：

$$
\mu = \frac{1}{B \cdot N} \sum_{b=1}^{B}\sum_{n=1}^{N} x_{b,n}, \quad \sigma^2 = \frac{1}{B \cdot N} \sum_{b=1}^{B}\sum_{n=1}^{N} (x_{b,n} - \mu)^2
$$

$$
\hat{x} = \frac{x - \mu}{\sqrt{\sigma^2 + \epsilon}}, \quad y = \gamma \hat{x} + \beta
$$

**缺点**：

- 对 batch size 敏感，小 batch 时统计量不稳定
- 训练和推理行为不一致（需要维护 running mean/var）
- RNN 中逐时间步统计不一致

**代码：**

```python
class BatchNorm(nn.Module):
    def __init__(self, num_features, eps=1e-5, momentum=0.1):
        super().__init__()
        self.eps = eps
        self.momentum = momentum
        self.gamma = nn.Parameter(torch.ones(num_features))
        self.beta = nn.Parameter(torch.zeros(num_features))
        self.register_buffer('running_mean', torch.zeros(num_features))
        self.register_buffer('running_var', torch.ones(num_features))

    def forward(self, x):
        B, N, D = x.shape
        x = x.view(-1, D)

        if self.training:
            mean = x.mean(dim=0)
            var = x.var(dim=0, unbiased=False)

            self.running_mean = (1 - self.momentum) * self.running_mean + self.momentum * mean
            n = x.shape[0]
            if n > 1:
                self.running_var = (1 - self.momentum) * self.running_var + self.momentum * var * n / (n - 1)
        else:
            mean = self.running_mean
            var = self.running_var

        x = (x - mean) / torch.sqrt(var + self.eps)
        out = self.gamma * x + self.beta
        return out.view(B, N, D)
```

---

## Layer Normalization

对特征维度做归一化，不依赖 batch 统计。给定输入 $x \in \mathbb{R}^{B \times N \times D}$，在每个 $(b, n)$ 位置的 D 维度求均值和方差：

$$
\mu_{b,n} = \frac{1}{D} \sum_{d=1}^{D} x_{b,n,d}, \quad \sigma^2_{b,n} = \frac{1}{D} \sum_{d=1}^{D} (x_{b,n,d} - \mu_{b,n})^2
$$

**优点**：

- 与 batch size 无关，训练和推理一致
- 适合 RNN / Transformer

**代码：**

```python
class LayerNorm(nn.Module):
    def __init__(self, num_features, eps=1e-5):
        super().__init__()
        self.eps = eps
        self.gamma = nn.Parameter(torch.ones(num_features))
        self.beta = nn.Parameter(torch.zeros(num_features))

    def forward(self, x):
        mean = x.mean(dim=-1, keepdim=True)
        var = x.var(dim=-1, keepdim=True, unbiased=False)
        x = (x - mean) / torch.sqrt(var + self.eps)
        return self.gamma * x + self.beta
```

---

## RMS Normalization

LayerNorm 的简化版，去掉了均值中心化和偏置项 $\beta$，仅保留缩放参数 $\gamma$，用 root mean square 替代标准差：

$$
\text{RMS}(x) = \sqrt{\frac{1}{D} \sum_{d=1}^{D} x_d^2}
$$

$$
y = \frac{x}{\text{RMS}(x)} \cdot \gamma
$$

**优点**：

- 比 LayerNorm 少一次 mean 计算，速度更快。LLaMA 等模型广泛使用。

**代码：**

```python
class RMSNorm(nn.Module):
    def __init__(self, num_features, eps=1e-5):
        super().__init__()
        self.eps = eps
        self.gamma = nn.Parameter(torch.ones(num_features))

    def forward(self, x):
        rms = torch.sqrt(x.pow(2).mean(dim=-1, keepdim=True) + self.eps)
        x = x / rms
        return self.gamma * x
```

---

## QK Normalization

专用于 Transformer 注意力机制。在计算 attention score 之前，对 Query 和 Key 分别做 LayerNorm 或 RMSNorm，再乘以一个可学习的缩放系数 $\alpha$：

$$
\hat{Q} = \text{Norm}(Q), \quad \hat{K} = \text{Norm}(K)
$$

$$
\text{Attention}(Q, K, V) = \text{softmax}\!\left(\alpha \cdot \hat{Q} \hat{K}^\top\right) V
$$

其中 $\alpha$ 是可学习的标量参数（per-head），用于控制归一化后 attention logits 的尺度。

**动机**：

训练大模型时，Q/K 的模长可能剧烈增长，导致 softmax 退化为 one-hot，梯度消失。例如：

$$
\begin{aligned}
\text{softmax}([760, 752, 750])
&= \text{softmax}([12, 4, 2]) \\
&= [0.99962, 0.00034, 0.00005]
\end{aligned}
$$

退化为 one-hot，注意力只能关注单个 token，丧失了捕捉多样上下文的能力。
QK-Norm 将 Q、K 约束在单位超球面上，抑制 $QK^\top$ 内积的绝对值膨胀。

$\alpha$ 恢复合理的注意力分布，稳定训练。 实践中工程上常直接用 $\alpha = 1/\sqrt{d_k}$ 固定值，退化为标准 Scaled Dot-Product Attention 的缩放系数，无需引入额外可学习参数，实现更简单。
