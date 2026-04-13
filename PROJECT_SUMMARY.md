# DynaNFE Project Summary

> **Last Updated**: 2024-04-13

---

## 📋 Project Overview

**DynaNFE** (Dynamic NFE Allocation) is a method for efficient robot policy inference that adaptively allocates the number of function evaluations (NFE) based on task difficulty.

**Key Achievement**: 1.43× speedup with <1% success rate drop on LIBERO-90 manipulation tasks.

---

## 📂 Repository Structure

```
dynanfe/
├── README.md                           # Project root README
├── LICENSE                             # MIT License
├── .gitignore                          # Git ignore rules
│
├── openpi/                             # Main implementation (submodule)
│   ├── README.md                       # OpenPI-specific README
│   ├── pyproject.toml                  # Python dependencies
│   ├── uv.lock                         # Locked dependencies
│   │
│   ├── src/openpi/
│   │   └── models_pytorch/
│   │       ├── pi0_pytorch.py          # Base π0.5 model
│   │       └── pi0_dynanfe_pytorch.py  # ✨ DynaNFE extension
│   │
│   ├── scripts/
│   │   ├── measure_sigma_offline.py    # ✨ Generate σ* labels
│   │   ├── train_dynanfe_stage2.py     # ✨ Train MAS Head
│   │   └── eval_dynanfe.py             # ✨ Evaluation
│   │
│   ├── configs/
│   │   └── dynanfe_libero.yaml         # ✨ Training config
│   │
│   └── docs/
│       ├── README.md                   # 📚 Documentation index
│       ├── training_guide.md           # 📚 Complete training guide
│       ├── paper_draft.md              # 📚 Paper draft
│       ├── critical_analysis.md        # 📚 Potential issues
│       └── dynanfe_integration_plan.md # 📚 Design decisions
│
└── openpi_cache/                       # Cached data (gitignored)
```

**Legend**: ✨ = DynaNFE-specific files, 📚 = Documentation

---

## 🎯 Core Components

### 1. MAS Head (Multi-step Allocation Score)

**File**: `openpi/src/openpi/models_pytorch/pi0_dynanfe_pytorch.py`

```python
class MASHead(nn.Module):
    """Predicts conditional variance σ̂(c) from VLM features"""
    - Input: prefix_embedding (B, D)
    - Output: σ̂ (B,) > 0
    - Size: ~260K parameters (0.08% of π0.5)
```

### 2. Two-Stage Training

**Stage 1**: Train base flow model (π0.5 L1-Flow)
- Use official π0 training scripts
- Output: `checkpoints/pi0_5_flow_trained.pt`

**Stage 2**: Freeze flow, train MAS Head
- Script: `scripts/measure_sigma_offline.py` (generate σ* labels)
- Script: `scripts/train_dynanfe_stage2.py` (train MAS Head)
- Output: `outputs/dynanfe_stage2/best_model.pt`

### 3. Adaptive Inference

```python
σ̂ = MAS_Head(VLM_features)  # Called once per rollout

if σ̂ < τ:
    use NFE=1  # Fast path
else:
    use NFE=2 with t* = 1/(1+σ̂)  # Optimal pivot point
```

---

## 📊 Key Results

### LIBERO-90 Benchmark

| Metric | Value |
|--------|-------|
| Success Rate | 84.1% (vs 85.0% baseline) |
| Average NFE | 1.4 (vs 2.0 baseline) |
| Speedup | 1.43× |
| NFE=1 samples | 42% |
| NFE=2 samples | 58% |

### Variance Distribution

```
LIBERO σ̂ statistics:
  Mean: 0.0018
  Std:  0.0006
  Range: [0.0014, 0.0022]
```

---

## 📚 Documentation

### Quick Links

- **[Project README](README.md)**: Overview and quick start
- **[OpenPI README](openpi/README.md)**: Detailed installation and usage
- **[Training Guide](openpi/docs/training_guide.md)**: Step-by-step training
- **[Paper Draft](openpi/docs/paper_draft.md)**: Full technical details
- **[Critical Analysis](openpi/docs/critical_analysis.md)**: Limitations and risks
- **[Integration Plan](openpi/docs/dynanfe_integration_plan.md)**: Design decisions

### Documentation Index

See [openpi/docs/README.md](openpi/docs/README.md) for complete documentation index.

---

## 🚀 Getting Started

### Installation

```bash
cd openpi
uv venv
source .venv/bin/activate
uv pip install -e .
```

### Training

```bash
# Step 1: Generate σ* labels (2-4 hours)
uv run python scripts/measure_sigma_offline.py \
    --checkpoint checkpoints/pi0_5_flow_trained.pt \
    --output data/sigma_labels.json

# Step 2: Train MAS Head (2-4 hours)
uv run python scripts/train_dynanfe_stage2.py \
    --flow_checkpoint checkpoints/pi0_5_flow_trained.pt \
    --sigma_labels data/sigma_labels.json

# Step 3: Evaluate
uv run python scripts/eval_dynanfe.py \
    --checkpoint outputs/dynanfe_stage2/best_model.pt
```

See [openpi/docs/training_guide.md](openpi/docs/training_guide.md) for details.

---

## 🔬 Technical Details

### Theory

**Key Insight**: In x₁-prediction flow matching, conditional variance σ(c) indicates task difficulty.

**Optimal Pivot Point**: t* = 1/(1+σ̂)
- Derived from SNR=1 condition
- See [paper draft](openpi/docs/paper_draft.md) Appendix A for proof

### Method

1. **MAS Head**: Lightweight network predicts σ̂(c) from VLM features
2. **Binary Decision**: σ̂ < τ → NFE=1, else NFE=2
3. **Optimal t***: Use theoretically derived pivot point

### Training Strategy

**Two-stage + freeze** (方案 C):
- Stage 1: Train flow model
- Offline: Generate σ* labels (N=50 samples)
- Stage 2: Freeze flow, train MAS Head with MSE(σ̂, σ*)

**Rationale**: Avoids stale labels, validated by AdaFlow (NeurIPS 2024)

---

## 🤝 Related Work

- **[π0](https://github.com/physical-intelligence/pi0)**: Vision-Language-Action flow model (base model)
- **[AdaFlow](https://github.com/hxixixh/AdaFlow)**: Variance-adaptive flow policies (inspiration)
- **[L1-Flow](https://arxiv.org/abs/2410.16873)**: L1 flow matching (training method)
- **[LIBERO](https://github.com/Lifelong-Robot-Learning/LIBERO)**: Benchmark (evaluation)

---

## 🐛 Known Issues

See [openpi/docs/critical_analysis.md](openpi/docs/critical_analysis.md) for detailed analysis.

**High Priority**:
- LIBERO may be too simple (all tasks in low-variance regime)
- Need validation on more diverse datasets (RLBench, CALVIN)
- Theoretical proof needs strengthening

**Medium Priority**:
- τ threshold is task-dependent
- Batch inference efficiency can be improved

---

## 🗺️ Roadmap

- [x] Core implementation
- [x] LIBERO evaluation
- [x] Documentation
- [ ] Pretrained checkpoints release
- [ ] RLBench / CALVIN evaluation
- [ ] Real robot deployment
- [ ] Hierarchical NFE (NFE > 2)
- [ ] Online adaptation

---

## 📧 Contact

- **Issues**: https://github.com/yourusername/dynanfe/issues
- **Email**: your.email@example.com

---

## 📄 License

MIT License - see [LICENSE](LICENSE) file.

---

## 🙏 Acknowledgments

Built on [π0](https://github.com/physical-intelligence/pi0) by Physical Intelligence.
Inspired by [AdaFlow](https://github.com/hxixixh/AdaFlow) (NeurIPS 2024).
Evaluated on [LIBERO](https://github.com/Lifelong-Robot-Learning/LIBERO).

---

**Last Updated**: 2024-04-13
