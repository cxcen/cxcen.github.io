---
title: Metis DEX Aggregator 路由算法
type: post
categories:
  - algorithm
  - dex
tags:
  - dex
  - algorithm
---

## 概述

Metis 是 Jupiter v3 中引入的一种专为 DEX 聚合设计的路由算法，它基于经典的 Bellman-Ford 算法进行了重大改进，旨在在多个 DEX 平台之间找到最优的代币交换路径。

## 核心目标

与传统的图算法不同，Metis 的目标不是找到最短路径，而是找到能够**最大化输出代币数量**的路径。这使其特别适合 DEX 聚合场景，因为用户关心的是最终能获得多少目标代币，而不是路径的长度。

## 算法基础

### 经典 Bellman-Ford 算法回顾

Bellman-Ford 算法用于在带权图中找到从源点到所有其他点的最短路径：

```rust
// 经典 Bellman-Ford 伪代码
for i in 1..V-1:
    for each edge (u, v) with weight w:
        if dist[u] + w < dist[v]:
            dist[v] = dist[u] + w
            predecessor[v] = u
```

### Metis 的关键改进

#### 1. 权重重新定义

**传统算法**：权重表示路径成本（距离）
**Metis 算法**：权重表示汇率的负对数

```rust
// Metis 权重计算
weight = -log(exchange_rate)
```

**原理**：
- 汇率是乘法关系：`最终数量 = 初始数量 × 汇率1 × 汇率2 × ...`
- 通过取对数转换为加法关系：`log(最终数量) = log(初始数量) + log(汇率1) + log(汇率2) + ...`
- 取负值是为了将最大化问题转换为最小化问题

#### 2. 价值最大化而非路径最小化

```rust
// 传统目标：最小化路径权重
minimize: Σ(edge_weights)

// Metis 目标：最大化输出代币
maximize: input_amount × Π(exchange_rates)
```

## 算法架构

### 数据结构

#### 1. 图表示

```rust
pub struct RoutingGraph {
    pub nodes: HashMap<String, Token>,        // 代币节点
    pub edges: HashMap<String, Vec<Edge>>,    // 交易对边
    pub config: RouterConfig,                 // 配置参数
}
```

#### 2. 边权重计算

```rust
pub struct Edge {
    pub exchange_rate: Decimal,    // 当前汇率
    pub liquidity: Decimal,        // 可用流动性
    pub max_trade_size: Decimal,   // 最大交易规模
    pub weight: f64,              // -log(exchange_rate)
}
```

### 核心算法流程

#### 1. 初始化阶段

```rust
fn initialize_nodes(&self, start_token: &str) -> Result<HashMap<String, GraphNode>> {
    let mut nodes = HashMap::new();
    
    for (addr, token) in &self.nodes {
        nodes.insert(addr.clone(), GraphNode {
            token: token.clone(),
            distance: f64::INFINITY,      // 初始距离为无穷大
            predecessor: None,            // 无前驱节点
            best_amount: dec!(0),         // 最优数量为 0
            liquidity_used: dec!(0),      // 已使用流动性为 0
        });
    }
    
    // 设置起始节点
    if let Some(start_node) = nodes.get_mut(&start_addr) {
        start_node.distance = 0.0;                    // 起始距离为 0
        start_node.best_amount = request.input_amount; // 起始数量
    }
    
    Ok(nodes)
}
```

#### 2. 松弛操作（Relaxation）

这是算法的核心，Metis 的松弛操作包含了多个约束：

```rust
async fn relax_edge(
    &self,
    state: &mut IterationState,
    edge: &Edge,
    request: &RouteRequest,
) -> Result<()> {
    let from_addr = &edge.from_token.address;
    let to_addr = &edge.to_token.address;
    
    if let Some(from_node) = state.nodes.get(from_addr) {
        if from_node.distance == f64::INFINITY {
            return Ok(()); // 跳过不可达节点
        }

        // 1. 计算潜在改进
        let new_distance = from_node.distance + edge.weight;
        let potential_amount = from_node.best_amount * edge.exchange_rate;
        
        // 2. 应用流动性约束
        let available_liquidity = edge.liquidity - edge.max_trade_size.min(edge.liquidity);
        let constrained_amount = potential_amount.min(available_liquidity);
        
        // 3. 检查此路径是否更好
        if let Some(to_node) = state.nodes.get_mut(to_addr) {
            if new_distance < to_node.distance && constrained_amount > dec!(0) {
                // 4. 额外约束检查
                if constrained_amount >= edge.min_trade_size 
                   && self.calculate_price_impact(edge, constrained_amount) <= self.config.max_price_impact {
                    
                    // 5. 更新节点状态
                    to_node.distance = new_distance;
                    to_node.predecessor = Some(from_addr.clone());
                    to_node.best_amount = constrained_amount;
                    to_node.liquidity_used = constrained_amount;
                    
                    state.improved = true;
                }
            }
        }
    }
    
    Ok(())
}
```

#### 3. 主循环

```rust
pub async fn find_optimal_route(&self, request: &RouteRequest) -> Result<Option<Route>> {
    // 初始化节点
    let mut nodes = self.initialize_nodes(&request.input_token)?;
    
    let mut iteration_state = IterationState {
        nodes,
        improved: true,
        iteration: 0,
        best_route: None,
    };

    // 增强 Bellman-Ford 迭代
    while iteration_state.improved 
          && iteration_state.iteration < request.max_iterations {
        
        iteration_state.improved = false;
        iteration_state.iteration += 1;
        
        // 处理所有边
        for (from_addr, edges) in &self.edges {
            if let Some(_from_node) = iteration_state.nodes.get(from_addr) {
                for edge in edges {
                    self.relax_edge(&mut iteration_state, edge, request).await?;
                }
            }
        }

        // 早期终止
        if !iteration_state.improved {
            break;
        }
    }

    // 提取最优路由
    self.extract_route(&iteration_state, request)
}
```

## 关键约束机制

### 1. 流动性约束

```rust
// 可用流动性计算
let available_liquidity = edge.liquidity - edge.max_trade_size.min(edge.liquidity);
let constrained_amount = potential_amount.min(available_liquidity);
```

**作用**：
- 防止大额交易导致的价格滑点
- 确保交易在 DEX 的承受范围内
- 模拟真实的交易执行环境

### 2. 价格影响约束

```rust
fn calculate_price_impact(&self, edge: &Edge, trade_amount: Decimal) -> Decimal {
    let impact_ratio = trade_amount / edge.liquidity;
    impact_ratio * dec!(0.5) // 简化的线性模型
}
```

**作用**：
- 计算交易对价格的影响程度
- 拒绝价格影响过大的路由
- 保护用户免受高滑点损失

### 3. 最小交易规模约束

```rust
if constrained_amount >= edge.min_trade_size {
    // 继续处理
}
```

**作用**：
- 确保交易规模满足 DEX 要求
- 避免过小的无效交易

## 路径提取

### 1. 路径重建

```rust
fn extract_route(&self, state: &IterationState, request: &RouteRequest) -> Result<Option<Route>> {
    let output_addr = self.get_token_address(&request.output_token)?;
    
    if let Some(output_node) = state.nodes.get(&output_addr) {
        if output_node.distance == f64::INFINITY {
            return Ok(None); // 未找到路径
        }

        // 重建路径
        let mut segments = Vec::new();
        let mut current_addr = output_addr.clone();
        let mut current_amount = output_node.best_amount;

        while let Some(predecessor_addr) = &state.nodes[&current_addr].predecessor {
            let edge = self.find_edge(predecessor_addr, &current_addr)?;
            let predecessor_node = &state.nodes[predecessor_addr];
            
            // 创建路径段
            segments.push(PathSegment {
                from_token: edge.from_token.clone(),
                to_token: edge.to_token.clone(),
                dex_platform: edge.dex_platform.clone(),
                input_amount: predecessor_node.best_amount,
                output_amount: current_amount,
                exchange_rate: current_amount / predecessor_node.best_amount,
                price_impact: self.calculate_price_impact(edge, predecessor_node.best_amount),
            });

            current_addr = predecessor_addr.clone();
            current_amount = predecessor_node.best_amount;
        }

        segments.reverse(); // 反转以获得正确顺序
        
        // 构建最终路由
        Ok(Some(Route {
            segments,
            total_input_amount: request.input_amount,
            total_output_amount: segments.last().unwrap().output_amount,
            effective_rate: segments.last().unwrap().output_amount / request.input_amount,
            price_impact: segments.iter().map(|s| s.price_impact).sum(),
            gas_estimate: self.estimate_gas_cost(&segments),
            split_ratio: None,
        }))
    } else {
        Ok(None)
    }
}
```

## 分割路由机制

### 1. 分割策略

对于大额交易，Metis 支持将交易分割到多个 DEX：

```rust
pub async fn find_split_routes(&self, request: &RouteRequest) -> Result<Option<SplitRoute>> {
    let mut split_routes = Vec::new();
    let max_splits = request.max_splits.unwrap_or(3);

    // 尝试找到具有不同数量的多个路由
    for split_idx in 0..max_splits {
        // 计算分割数量（递减部分）
        let split_ratio = if split_idx == 0 { dec!(0.6) } 
                         else if split_idx == 1 { dec!(0.3) } 
                         else { dec!(0.1) };
        
        let split_amount = request.input_amount * split_ratio;
        
        if split_amount < dec!(10) { // 最小可行数量
            break;
        }

        let mut split_request = request.clone();
        split_request.input_amount = split_amount;

        if let Some(route) = self.find_optimal_route(&split_request).await? {
            split_routes.push(route);
        }
    }

    // 计算组合指标
    let total_input = split_routes.iter().map(|r| r.total_input_amount).sum();
    let total_output = split_routes.iter().map(|r| r.total_output_amount).sum();
    let effective_rate = total_output / total_input;

    Ok(Some(SplitRoute {
        routes: split_routes,
        total_input_amount: total_input,
        total_output_amount: total_output,
        effective_rate,
        price_impact: split_routes.iter().map(|r| r.price_impact).sum(),
        gas_estimate: split_routes.iter().map(|r| r.gas_estimate).sum(),
    }))
}
```

### 2. 分割优势

- **减少价格影响**：大额交易分散到多个 DEX
- **提高流动性利用率**：利用不同 DEX 的流动性池
- **降低滑点**：避免单一 DEX 的深度不足

## 性能优化

### 1. 早期终止

```rust
// 当没有改进时提前终止
if !iteration_state.improved {
    debug!("✅ 迭代 {} 中没有改进，提前终止", iteration_state.iteration);
    break;
}
```

### 2. 迭代次数限制

```rust
while iteration_state.improved 
      && iteration_state.iteration < request.max_iterations {
    // 处理逻辑
}
```

### 3. 流动性剪枝

```rust
// 跳过流动性不足的边
if constrained_amount < edge.min_trade_size {
    continue;
}
```

## 算法复杂度分析

### 时间复杂度

- **传统 Bellman-Ford**：O(V × E)
- **Metis 算法**：O(k × E)，其中 k 是实际迭代次数
- **平均情况**：k << V，因为早期终止和约束剪枝

### 空间复杂度

- **节点存储**：O(V)
- **边存储**：O(E)
- **总空间**：O(V + E)

## 实际应用示例

### 示例：USDC → SOL 路由

```rust
// 输入：1000 USDC
let request = RouteRequest {
    input_token: "USDC".to_string(),
    output_token: "SOL".to_string(),
    input_amount: dec!(1000.0),
    slippage_tolerance: dec!(0.005), // 0.5%
    max_iterations: 5,
    enable_split_routes: true,
    max_splits: Some(3),
};

// 可能的路径：
// 路径 1: USDC → RAY → SOL (通过 Raydium + Orca)
// 路径 2: USDC → SOL (直接通过 Raydium)
// 路径 3: 分割路由 (60% 路径1 + 30% 路径2 + 10% 其他)
```

### 算法执行过程

1. **初始化**：所有节点距离设为 ∞，USDC 节点距离设为 0
2. **迭代 1**：处理 USDC → RAY 边，更新 RAY 节点
3. **迭代 2**：处理 RAY → SOL 边，更新 SOL 节点
4. **路径提取**：从 SOL 节点回溯到 USDC 节点
5. **结果**：找到最优路径 USDC → RAY → SOL

[Github: https://github.com/cxcen/metis](https://github.com/cxcen/metis)

