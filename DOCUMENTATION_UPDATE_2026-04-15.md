# DynaNFE 文档重构总结

> **日期**: 2026-04-15
> **目的**: 解决 σ 归一化问题，重新梳理理论框架

---

## 核心问题

用户提出的关键问题：
1. **σ 依赖归一化**: σ(c) 的绝对值依赖于动作归一化，在不同数据集上不可比
2. **阈值 τ 难以设置**: 在简单数据集（如 LIBERO）上，所有 σ(c) 都很小，τ 的设置成为问题
3. **t* 公式的理论基础**: 为什么 t* 的不同取值会影响结果？

## 解决方案

### 1. 引入 σ_marginal 作为参考尺度

```python
σ_marginal = std(x₁)  # 整个训练集的边际标准差
σ_eff(c) = σ(c) / σ_marginal  # 有效方差（无量纲）
```

**修正后的公式**:
```
t* = σ_marginal / (σ_marginal + σ(c)) = 1 / (1 + σ_eff(c))
决策: σ_eff(c) < τ → NFE=1
```

**优势**:
- τ 现在是无量纲比例（如 0.1 表示"条件方差 < 边际的 10%"）
- t* 公式在不同数据集上可比
- 有明确的物理含义

### 2. 创建理论推导文档

新文档 `theory_derivation.md` 详细解释：
- 为什么 t* 的取值影响结果（信噪比、模式选择）
- SNR=1 条件的物理意义
- σ_marginal 的三种选法对比
- 多模态分布的模式选择机制

### 3. 更新所有相关文档

**已更新的文档**:
1. ✅ `theory_derivation.md` - 新建，完整理论推导
2. ✅ `dynanfe_integration_plan.md` - 更新实现方案
3. ✅ `training_guide.md` - 添加步骤 0（计算 σ_marginal）
4. ✅ `paper_draft.md` - 更新理论部分和 Appendix A
5. ✅ `critical_analysis.md` - 更新问题分析

---

## 主要变更

### 理论框架

**之前**:
```
t* = 1 / (1 + σ(c))
决策: σ(c) < τ → NFE=1
问题: σ(c) 和 τ 都依赖归一化
```

**之后**:
```
σ_marginal = std(x₁)  # 离线计算一次
σ_eff(c) = σ(c) / σ_marginal
t* = 1 / (1 + σ_eff(c))
决策: σ_eff(c) < τ → NFE=1
优势: τ 是无量纲比例，可比
```

### 训练流程

**新增步骤 0**:
```bash
# 计算 σ_marginal（一次性）
python scripts/compute_sigma_marginal.py \
    --dataset libero \
    --output data/sigma_marginal.json
# 输出: {"sigma_marginal": 0.152, ...}
```

**步骤 2 更新**:
```bash
# 训练时传入 σ_marginal
python scripts/train_dynanfe_stage2.py \
    --sigma_marginal 0.152 \
    ...
```

### 推理协议

**之前**:
```python
σ̂ = MAS_Head(c)
if σ̂ < τ:
    NFE = 1
else:
    t* = 1 / (1 + σ̂)
```

**之后**:
```python
σ̂ = MAS_Head(c)
σ̂_eff = σ̂ / σ_marginal
if σ̂_eff < τ:
    NFE = 1
else:
    t* = σ_marginal / (σ_marginal + σ̂)
```

### 超参数含义

**τ 的物理含义**（现在是无量纲的）:
- τ = 0.05: 条件方差 < 边际的 5%（激进）
- τ = 0.10: 条件方差 < 边际的 10%（均衡）
- τ = 0.20: 条件方差 < 边际的 20%（保守）

---

## 数值例子（LIBERO）

假设 σ_marginal = 0.15:

| 样本类型 | σ(c) | σ_eff | t* | 决策 (τ=0.1) |
|---------|------|-------|-----|-------------|
| 简单直行 | 0.001 | 0.007 | 0.993 | NFE=1 |
| 一般操作 | 0.005 | 0.033 | 0.968 | NFE=1 |
| 复杂抓取 | 0.03 | 0.200 | 0.833 | NFE=2 |
| 高度模糊 | 0.10 | 0.667 | 0.600 | NFE=2 |

现在有明显的区分度！

---

## 文档结构

```
openpi/docs/
├── README.md                       # 文档索引
├── theory_derivation.md            # ✨ 新建：完整理论推导
├── dynanfe_integration_plan.md     # ✅ 更新：实现方案
├── training_guide.md               # ✅ 更新：训练流程
├── paper_draft.md                  # ✅ 更新：论文草稿
└── critical_analysis.md            # ✅ 更新：问题分析
```

---

## 下一步

### 代码实现

需要实现的脚本：
1. `scripts/compute_sigma_marginal.py` - 计算 σ_marginal
2. 更新 `scripts/measure_sigma_offline.py` - 同时输出 σ_eff
3. 更新 `scripts/train_dynanfe_stage2.py` - 接受 σ_marginal 参数
4. 更新 `src/openpi/models_pytorch/pi0_dynanfe_pytorch.py` - 使用 σ_eff

### 实验验证

1. **验证 σ_marginal 的选择**: 对比边际 std vs 其他参考尺度
2. **验证 t* 公式**: 对比理论公式 vs 固定 t=0.5 vs 网格搜索
3. **调优 τ**: 找到最佳 success rate / speedup 平衡点

### 论文修改

1. 更新 Introduction 和 Method 部分
2. 补充 Appendix A 的完整推导
3. 添加 σ_marginal 的消融实验

---

## 理论贡献

相比原始方案的改进：

| 方面 | 原始方案 | 改进方案 |
|------|---------|---------|
| 阈值 τ | 绝对值，数据集相关 | 无量纲比例，可比 |
| t* 公式 | t* = 1/(1+σ) | t* = 1/(1+σ_eff) |
| 物理含义 | 不清晰 | "条件化后剩余的不确定性比例" |
| 跨数据集 | 不可比 | 可比 |

---

## 记忆更新

已保存到记忆系统：
- `feedback_sigma_normalization.md` - σ 归一化问题的解决方案

---

**状态**: ✅ 文档重构完成，理论框架更新完毕
