---
title: AMM中x_N.y=k曲线的数学公式推导
type: post
categories:
  - AMM
  - uniswap
tags:
  - AMM
  - math

---

<head>
   <script type="text/javascript" async
      src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js">
   </script>
   <script>
      MathJax = {
        tex: {
          inlineMath: [['$', '$'], ['$`', '`$'], ['\\(', '\\)']],
          displayMath: [['```math', '```'], ['$$', '$$'], ['\\[', '\\]']]
        }
      };
   </script>
</head>

## $x^N \cdot y = k$ 曲线AMM数学推导


## 第一部分：基础理论

### 1.1 不变量定义

设流动性池包含两种代币：
- 代币 X：数量为 $x$
- 代币 Y：数量为 $y$

定义不变量函数：
$$
\boxed{x^N \cdot y = k}
$$

其中：
- $N \in \mathbb{N}^+$：曲线参数（$N=1$ 时退化为经典 Uniswap 模型）
- $k > 0$：流动性常数

### 1.2 瞬时价格推导

价格定义为边际汇率，即 Y 代币相对于 X 代币的价格：$$P = -\frac{dy}{dx}$$

对不变量进行全微分：$$d(x^N \cdot y) = 0$$
根据乘积的微分法则：$$d(x^N \cdot y) = y \cdot d(x^N) + x^N \cdot dy = 0$$
对 $x^N$ 求导：$$d(x^N) = N \cdot x^{N-1} \cdot dx$$
代入得：$$N \cdot x^{N-1} \cdot y \cdot dx + x^N \cdot dy = 0$$
解得：$$\frac{dy}{dx} = -\frac{N \cdot x^{N-1} \cdot y}{x^N} = -\frac{N \cdot y}{x}$$
因此瞬时价格为：
$$
\boxed{P = \frac{N \cdot y}{x}}
$$

结合不变量 $y = \frac{k}{x^N}$，代入价格公式：$$P = \frac{N \cdot y}{x} = \frac{N \cdot k/x^N}{x} = \frac{N \cdot k}{x^{N+1}}$$
得到价格的另一表达式：
$$
\boxed{P = \frac{N \cdot k}{x^{N+1}}}
$$

### 1.3 储备量与价格的关系（引入流动性常数 L）

从 $P = \frac{N \cdot k}{x^{N+1}}$ 反解：$$x = \left(\frac{N \cdot k}{P}\right)^{\frac{1}{N+1}}$$
再由 $y = \frac{k}{x^N}$ 得：$$y = k \cdot \left(\frac{P}{N \cdot k}\right)^{\frac{N}{N+1}} = k^{\frac{1}{N+1}} \cdot N^{-\frac{N}{N+1}} \cdot P^{\frac{N}{N+1}}$$
为便于记忆与运算，定义"流动性常数" $\boxed{L \;\triangleq\; (N \cdot k)^{\frac{1}{N+1}}}$。则有等价而简洁的表示：
$$
\boxed{x(P) = L \cdot P^{-\frac{1}{N+1}}} \quad\text{与}\quad \boxed{y(P) = \frac{L}{N} \cdot P^{\frac{N}{N+1}}}
$$
并且二者与 $k$ 的关系：
$$
\boxed{k = \frac{L^{N+1}}{N}}
$$

---

## 第二部分：集中流动性理论

### 2.1 价格区间与虚拟储备

考虑价格区间 $[P_{\min}, P_{\max}]$，其中 $0 < P_{\min} < P_{\max}$。定义区间对应的"虚拟储备"变化量：
- 虚拟 X 储备（需要的 X 数量）：$\;x_v = x(P_{\min}) - x(P_{\max})$
- 虚拟 Y 储备（需要的 Y 数量）：$\;y_v = y(P_{\max}) - y(P_{\min})$

由上节的 $x(P), y(P)$ 直接代入：
$$
\boxed{x_v = L \cdot \Big(P_{\min}^{-\tfrac{1}{N+1}} - P_{\max}^{-\tfrac{1}{N+1}}\Big)}
$$
$$
\boxed{y_v = \frac{L}{N} \cdot \Big(P_{\max}^{\tfrac{N}{N+1}} - P_{\min}^{\tfrac{N}{N+1}}\Big)}
$$

注意：以上是"净变化量"的定义，保持与常见的区间仓位记账一致。

### 2.2 流动性 L 的定义与与 k 的关系

由定义 $L = (N k)^{\tfrac{1}{N+1}}$ 可反推出 $k = L^{N+1}/N$。这保证了在任一价格 $P$ 下：
- $x(P) = L\,P^{-1/(N+1)}$
- $y(P) = (L/N)\,P^{N/(N+1)}$
且对任意 $P$ 有 $x(P)^N\,y(P) = (L^{N}\,P^{-\tfrac{N}{N+1}})\cdot \big((L/N)\,P^{\tfrac{N}{N+1}}\big) = L^{N+1}/N = k$，从而不变量恒成立。

进一步地，从区间虚拟储备反解出 $L$：
$$
\boxed{L = \frac{x_v}{P_{\min}^{-\tfrac{1}{N+1}} - P_{\max}^{-\tfrac{1}{N+1}}} = \frac{N\,y_v}{P_{\max}^{\tfrac{N}{N+1}} - P_{\min}^{\tfrac{N}{N+1}}}}
$$
该式与上面的 $L=(N k)^{1/(N+1)}$ 一致（可通过代入 $k$ 验证）。


### 2.3 储备量（注入量）计算公式与完整推导

给定 $L$ 和价格区间 $[P_{\min}, P_{\max}]$，为在该区间内提供集中流动性，需要的两种代币数量分别为：
$$
\boxed{\Delta x \;=\; L \cdot \Big(P_{\min}^{-\tfrac{1}{N+1}} - P_{\max}^{-\tfrac{1}{N+1}}\Big)}
$$

$$
\boxed{\Delta y \;=\; \frac{L}{N} \cdot \Big(P_{\max}^{\tfrac{N}{N+1}} - P_{\min}^{\tfrac{N}{N+1}}\Big)}
$$

推导要点：
- 由 $x(P)=L\,P^{-1/(N+1)}$ 对价格从 $P_{\max}$ 移到 $P_{\min}$ 的变化（X 端所需注入量）为差值 $x(P_{\min})-x(P_{\max})$。
- 由 $y(P)=(L/N)\,P^{N/(N+1)}$ 对价格从 $P_{\min}$ 移到 $P_{\max}$ 的变化（Y 端所需注入量）为差值 $y(P_{\max})-y(P_{\min})$。
- 二者与 $k$ 的等价关系可由 $k=L^{N+1}/N$ 验证。

### 2.4 由储备量反解流动性

$$
\boxed{L = \frac{\Delta x}{P_{\min}^{-\tfrac{1}{N+1}} - P_{\max}^{-\tfrac{1}{N+1}}}}
$$

$$
\boxed{L = \frac{N \cdot \Delta y}{P_{\max}^{\tfrac{N}{N+1}} - P_{\min}^{\tfrac{N}{N+1}}}}
$$
两式一致性可由 $k$ 的表达式交叉验证。

### 2.5 N=4 时的特殊考虑：Fifth Root 精度问题

当 $N=4$ 时，不变量为 $x^4 \cdot y = k$，价格函数为：
$$P = \frac{4 \cdot y}{x} = \frac{4 \cdot k}{x^5}$$

从价格反解储备量：
$$x(P) = L \cdot P^{-\frac{1}{5}} = L \cdot P^{-0.2}$$
$$y(P) = \frac{L}{4} \cdot P^{\frac{4}{5}} = \frac{L}{4} \cdot P^{0.8}$$


## 第三部分：交换机制（Swap）

记当前价格为 $P_1$，交换后价格为 $P_2$。核心恒等式：
$$
x(P) = L\,P^{-\tfrac{1}{N+1}}, \quad y(P) = \frac{L}{N}\,P^{\tfrac{N}{N+1}}
$$

### 3.1 精确输入：X → Y（卖出 X，得到 Y）

输入：$\Delta x_{in} > 0$。在无手续费理想情形下，池内 X 储备增加：
$$x_2 = x_1 + \Delta x_{in}$$
利用 $x = L\,P^{-1/(N+1)}$ 进行"线性化变量"替换 $u = P^{-1/(N+1)}$：
$$u_1 = P_1^{-\tfrac{1}{N+1}}, \quad u_2 = P_2^{-\tfrac{1}{N+1}}$$
于是
$$u_2 = u_1 + \frac{\Delta x_{in}}{L}$$
还原到价格：
$$
\boxed{P_2 = \big(P_1^{-\tfrac{1}{N+1}} + \tfrac{\Delta x_{in}}{L}\big)^{-(N+1)} = P_1 \cdot \Big(1 + \frac{\Delta x_{in}}{L} \cdot P_1^{\tfrac{1}{N+1}}\Big)^{-(N+1)}}
$$
输出端使用 $y(P)$ 的差值：
$$
\boxed{\Delta y_{out} = y_1 - y_2 = \frac{L}{N} \cdot \Big(P_1^{\tfrac{N}{N+1}} - P_2^{\tfrac{N}{N+1}}\Big)}
$$

### 3.2 精确输入：Y → X（卖出 Y，得到 X）

输入：$\Delta y_{in} > 0$。池内 Y 储备增加：
$$y_2 = y_1 + \Delta y_{in}$$
用"线性化变量"替换 $v = P^{N/(N+1)}$：
$$v_1 = P_1^{\tfrac{N}{N+1}}, \quad v_2 = P_2^{\tfrac{N}{N+1}}$$
由 $y = (L/N)\,v$ 得：
$$v_2 = v_1 + \frac{N}{L}\,\Delta y_{in}$$
还原到价格：
$$
\boxed{P_2 = \Big(P_1^{\tfrac{N}{N+1}} + \frac{N\,\Delta y_{in}}{L}\Big)^{\tfrac{N+1}{N}}}
$$
输出的 X 为正值（池内 X 减少）：
$$
\boxed{\Delta x_{out} = x_1 - x_2 = L \cdot \Big(P_1^{-\tfrac{1}{N+1}} - P_2^{-\tfrac{1}{N+1}}\Big)}
$$
> 说明：此前写成 $L\,(P_2^{-\tfrac{1}{N+1}} - P_1^{-\tfrac{1}{N+1}})$ 会给出负值，现已修正为正向输出定义。

### 3.3 精确输出：给定目标输出量求所需输入

- 期望获得 $\Delta y_{out}$ 的 Y（X → Y）：
  - 目标价格满足
    $$
    \boxed{P_2^{\tfrac{N}{N+1}} = P_1^{\tfrac{N}{N+1}} - \frac{N\,\Delta y_{out}}{L}}
    $$
  - 所需输入 X：
    $$
    \boxed{\Delta x_{in} = L \cdot \Big(P_1^{-\tfrac{1}{N+1}} - P_2^{-\tfrac{1}{N+1}}\Big)}
    $$

- 期望获得 $\Delta x_{out}$ 的 X（Y → X）：
  - 目标价格满足
    $$
    \boxed{P_2^{-\tfrac{1}{N+1}} = P_1^{-\tfrac{1}{N+1}} + \frac{\Delta x_{out}}{L}}
    $$
  - 所需输入 Y：
    $$
    \boxed{\Delta y_{in} = \frac{L}{N} \cdot \Big(P_2^{\tfrac{N}{N+1}} - P_1^{\tfrac{N}{N+1}}\Big)}
    $$

### 3.4 价格影响（Price Impact）

X → Y 的价格影响：
$$\text{PI} = \frac{P_2 - P_1}{P_1} = \Big(1 + \frac{\Delta x_{in}}{L} \cdot P_1^{\tfrac{1}{N+1}}\Big)^{-(N+1)} - 1$$
小额近似（$\Delta x_{in}$ 很小）：
$$\text{PI} \approx -(N+1) \cdot \frac{\Delta x_{in}}{L} \cdot P_1^{\tfrac{1}{N+1}}$$

---

## 第四部分：正确性与一致性校验

### 4.1 维度与符号
- $P$ 为"Y 对 X"的边际价格，定义为 $-dy/dx$，因此对 X→Y 交易（$\Delta x_{in}>0$）通常导致 $P$ 下降；对 Y→X 交易（$\Delta y_{in}>0$）通常导致 $P$ 上升。
- 所有输出量按正值定义：$\Delta y_{out} = y_1 - y_2$, $\Delta x_{out} = x_1 - x_2$。

### 4.2 退化到 $N=1$（恒定乘积）
- $L = (N k)^{1/(N+1)}$ 在 $N=1$ 时为 $L = \sqrt{k}$。
- $x(P) = L\,P^{-1/2},\; y(P) = L\,P^{1/2}$ 与 $x\,y=k$ 完全一致。
- X→Y：$P_2 = (P_1^{-1/2} + \Delta x/L)^{-2}$，与 CPAMM 的标准结果一致。

### 4.3 不变量保持
对任意 $P$：
$$x(P)^N y(P) = (L\,P^{-\tfrac{1}{N+1}})^N \cdot \Big(\tfrac{L}{N}\,P^{\tfrac{N}{N+1}}\Big) = \frac{L^{N+1}}{N} = k$$
交换前后状态沿曲线移动，因而不变量恒保持。

### 4.4 单调性与边界
- $x(P)$ 随 $P$ 单调递减，$y(P)$ 随 $P$ 单调递增。
- $P_{\min}, P_{\max} > 0$，且 $P_{\min} < P_{\max}$。
- 区间注入量 $\Delta x, \Delta y$ 非负。

### 4.5 一致性互证
- 用 $\Delta x$ 反解的 $L$ 与用 $\Delta y$ 反解的 $L$ 一致。
- 由目标输出反推的 $P_2$ 再代回输入公式，可复原原始输入量。

