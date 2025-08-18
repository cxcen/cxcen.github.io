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

#### 2.5.1 Fifth Root 表示方法

合约中记录的是价格的5次根（fifth root）：
$$\text{fifrt} = P^{\frac{1}{5}}$$

**重要说明**：合约内部只存储fifrt，且只使用fifrt的4次方进行计算。实际价格P是在外部计算的：
$$P = \text{fifrt}^5$$

存储格式：
- 高16位：fifrt的整数部分，最大$P $
- 低48位：fifrt的小数部分（定点数表示）

#### 2.5.2 计算策略与精度分析

**核心策略**：计算时直接将fifrt转换为u256，避免溢出问题

```rust
// 存储格式：u64，高16位整数，低48位小数
let fifrt: u64;  // 高16位：整数部分， 低48位：小数部分

// 计算策略：直接转换为u256进行计算
fn calculate_fifrt_fourth_safe(fifrt: u64) -> u256 {
    let fifrt_u256: u256 = fifrt.into();
    let fifrt_squared = fifrt_u256 * fifrt_u256;
    fifrt_squared * fifrt_squared  // 结果最大为 2^256，安全
}
```

#### 2.5.3 精度分配合理性分析

**当前16+48位设计的优势：**

1. **范围充足**：fifrt最大值约为$2^{64}$，满足大多数价格需求
2. **精度平衡**：48位小数提供约14位十进制精度
3. **计算安全**：转换为u256后，$\text{fifrt}^4$完全不会溢出
4. **存储效率**：单个u64存储，内存占用小

**数学验证：**

- 最大fifrt：约$2^{64}$
- $\text{fifrt}^4$最大值：约$2^{256}$，完全在u256范围内
- 最大价格P：约$2^{320}$，足够表示极端价格情况

#### 2.5.4 $\text{fifrt}^4$参与其他公式运算的溢出风险分析

基于文档中的数学公式，分析$\text{fifrt}^4$参与运算时的溢出风险：

**风险场景1：储备量计算中的溢出**

从公式：
$$x(P) = L \cdot P^{-\frac{1}{5}} = L \cdot \text{fifrt}^{-1}$$
$$y(P) = \frac{L}{4} \cdot P^{\frac{4}{5}} = \frac{L}{4} \cdot \text{fifrt}^4$$

计算 $y(P)$ 时：$y = \frac{L}{4} \cdot \text{fifrt}^4$

**溢出风险分析：**
- 如果 $L$ 很大（如 $2^{128}$），则 $\frac{L}{4} \cdot \text{fifrt}^4$ 可能超出 u256 范围
- 约束条件：$\frac{L}{4} \cdot \text{fifrt}^4 \leq \text{u256::MAX}$

**风险场景2：流动性L反解中的溢出**

从公式：
$$L = \frac{N \cdot y_v}{P_{\max}^{\frac{4}{5}} - P_{\min}^{\frac{4}{5}}} = \frac{4 \cdot y_v}{\text{fifrt}_{\max}^4 - \text{fifrt}_{\min}^4}$$

计算 $L$ 时：$L = \frac{4 \cdot y_v}{\text{fifrt}_{\max}^4 - \text{fifrt}_{\min}^4}$

**溢出风险分析：**
- 分母 $(\text{fifrt}_{\max}^4 - \text{fifrt}_{\min}^4)$ 最大约为 $2^{256}$
- 如果 $y_v$ 很大，$4 \cdot y_v$ 可能超出 u256
- 约束条件：$4 \cdot y_v \leq \text{u256::MAX}$

**风险场景3：交换计算中的溢出**

从公式：
$$\Delta y_{out} = \frac{L}{4} \cdot \left(P_1^{\frac{4}{5}} - P_2^{\frac{4}{5}}\right) = \frac{L}{4} \cdot \left(\text{fifrt}_1^4 - \text{fifrt}_2^4\right)$$

计算输出时：$\Delta y_{out} = \frac{L}{4} \cdot \left(\text{fifrt}_1^4 - \text{fifrt}_2^4\right)$

**溢出风险分析：**
- 如果 $L$ 很大，$L \cdot \left(\text{fifrt}_1^4 - \text{fifrt}_2^4\right)$ 可能超出 u256
- 约束条件：$L \cdot \left(\text{fifrt}_1^4 - \text{fifrt}_2^4\right) \leq 4 \cdot \text{u256::MAX}$

**风险场景4：价格影响计算中的溢出**

从公式：
$$\text{PI} \approx -5 \cdot \frac{\Delta x_{in}}{L} \cdot P_1^{\frac{1}{5}} = -5 \cdot \frac{\Delta x_{in}}{L} \cdot \text{fifrt}_1$$

计算价格影响时：$\text{PI} \approx -5 \cdot \frac{\Delta x_{in}}{L} \cdot \text{fifrt}_1$

**溢出风险分析：**
- 如果 $\Delta x_{in}$ 很大，$\Delta x_{in} \cdot \text{fifrt}_1$ 可能超出 u256
- 约束条件：$\Delta x_{in} \cdot \text{fifrt}_1 \leq \text{u256::MAX}$

**解决方案：分步计算与范围检查**
```rust
// 安全的储备量计算
fn safe_calculate_y_reserve(L: u256, fifrt: u64) -> Result<u256, Error> {
    let fifrt_fourth = calculate_fifrt_fourth_safe(fifrt);
    
    // 检查 (L/4) * fifrt^4 是否溢出
    if L > 0 && fifrt_fourth > u256::MAX / (L / 4) {
        return Err(Error::Overflow);
    }
    
    Ok((L / 4) * fifrt_fourth)
}

// 安全的流动性反解
fn safe_calculate_liquidity(y_v: u256, fifrt_max: u64, fifrt_min: u64) -> Result<u256, Error> {
    let fifrt_max_fourth = calculate_fifrt_fourth_safe(fifrt_max);
    let fifrt_min_fourth = calculate_fifrt_fourth_safe(fifrt_min);
    let denominator = fifrt_max_fourth - fifrt_min_fourth;
    
    // 检查 4 * y_v 是否超出u256
    if y_v > u256::MAX / 4 {
        return Err(Error::Overflow);
    }
    
    Ok((4 * y_v) / denominator)
}
```

**关键约束条件总结：**
为确保计算安全，需要满足：
1. $\frac{L}{4} \cdot \text{fifrt}^4 \leq \text{u256::MAX}$ - 储备量计算约束
2. $4 \cdot y_v \leq \text{u256::MAX}$ - 流动性反解约束  
3. $L \cdot \left(\text{fifrt}_1^4 - \text{fifrt}_2^4\right) \leq 4 \cdot \text{u256::MAX}$ - 交换计算约束
4. $\Delta x_{in} \cdot \text{fifrt} \leq \text{u256::MAX}$ - 价格影响计算约束

#### 2.5.5 推荐实现方案

**简化实现**：直接使用u256进行计算，无需复杂的溢出检查

```rust
// 推荐实现：直接转换为u256
fn safe_fifrt_fourth(fifrt: u64) -> u256 {
    let fifrt_u256: u256 = fifrt.into();
    let fifrt_squared = fifrt_u256 * fifrt_u256;
    fifrt_squared * fifrt_squared
}

// 辅助函数：提取整数和小数部分
fn extract_fifrt_parts(fifrt: u64) -> (u16, u64) {
    let integer = (fifrt >> 48) as u16;  // 高16位
    let fraction = fifrt & 0x0000_FFFF_FFFF_FFFF;  // 低48位
    (integer, fraction)
}

// 辅助函数：从整数和小数部分构造fifrt
fn construct_fifrt(integer: u16, fraction: u64) -> u64 {
    ((integer as u64) << 48) | (fraction & 0x0000_FFFF_FFFF_FFFF)
}
```

#### 2.5.6 总结

1. **实现简单**：直接转换为u256，无需溢出检查
2. **性能优秀**：避免了复杂的条件判断和回退逻辑
3. **安全可靠**：u256完全覆盖计算范围
4. **精度充足**：16+48位分配满足实际需求

---

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

