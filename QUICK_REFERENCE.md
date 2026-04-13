# DynaNFE Quick Reference Card

> **One-page reference for the entire project**

---

## 🎯 What is DynaNFE?

Adaptive inference for flow-based robot policies: **1.43× speedup** with <1% success rate drop.

**Core Idea**: Use a tiny neural network (MAS Head) to predict task difficulty and decide: NFE=1 (fast) or NFE=2 (accurate)?

---

## 📊 Key Numbers

| Metric | Value |
|--------|-------|
| Success Rate | 84% (vs 85% baseline) |
| Average NFE | 1.4 (vs 2.0 baseline) |
| Speedup | 1.43× |
| MAS Head Size | 260K params (0.08% of model) |
| NFE=1 samples | 42% |
| NFE=2 samples | 58% |

---

## 🚀 Quick Start (3 Commands)

```bash
# 1. Install
cd openpi && uv venv && source .venv/bin/activate && uv pip install -e .

# 2. Download model
wget https://huggingface.co/yourusername/dynanfe-libero/resolve/main/dynanfe_libero.pt

# 3. Run inference
python -c "from openpi.models_pytorch.pi0_dynanfe_pytorch import PI0PytorchWithDynaNFE; ..."
```

---

## 📚 Documentation Map

```
dynanfe/
├── README.md                    ← Start here (overview)
├── openpi/
│   ├── README.md                ← Installation & usage
│   └── docs/
│       ├── training_guide.md    ← How to train
│       ├── paper_draft.md       ← Full technical details
│       ├── critical_analysis.md ← Limitations & risks
│       └── dynanfe_integration_plan.md ← Design decisions
```

---

## 🔧 Training (3 Steps)

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

---

## 🧠 How It Works

```python
# 1. Predict variance
σ̂ = MAS_Head(VLM_features)  # Called once per rollout

# 2. Binary decision
if σ̂ < τ:
    use NFE=1  # Fast path (42% of samples)
else:
    use NFE=2 with t* = 1/(1+σ̂)  # Optimal pivot (58% of samples)
```

**Theory**: t* = 1/(1+σ̂) is derived from SNR=1 condition for optimal mode separation.

---

## 📁 Key Files

| File | Purpose |
|------|---------|
| `src/openpi/models_pytorch/pi0_dynanfe_pytorch.py` | Core implementation |
| `scripts/measure_sigma_offline.py` | Generate σ* labels |
| `scripts/train_dynanfe_stage2.py` | Train MAS Head |
| `docs/training_guide.md` | Training instructions |
| `docs/paper_draft.md` | Technical paper |
| `docs/critical_analysis.md` | Known issues |

---

## 🎓 Citation

```bibtex
@article{dynanfe2024,
  title={DynaNFE: Dynamic NFE Allocation for Efficient Robot Policy Inference},
  author={Your Name},
  journal={arXiv preprint arXiv:2024.XXXXX},
  year={2024}
}
```

---

## 🤝 Related Projects

- **π0**: Base model (Physical Intelligence)
- **AdaFlow**: Inspiration (NeurIPS 2024)
- **L1-Flow**: Training method (CoRL 2024)
- **LIBERO**: Benchmark (NeurIPS 2023)

---

## 🐛 Known Issues

**High Priority**:
- LIBERO may be too simple (need harder datasets)
- Theoretical proof needs strengthening

**Medium Priority**:
- τ threshold is task-dependent
- Batch inference can be optimized

See `docs/critical_analysis.md` for details.

---

## 🗺️ Roadmap

- [x] Core implementation
- [x] LIBERO evaluation
- [x] Documentation
- [ ] Pretrained checkpoints
- [ ] RLBench / CALVIN
- [ ] Real robot deployment

---

## 💡 Quick Tips

**New to the project?**
→ Read `README.md` (root)

**Want to train?**
→ Follow `docs/training_guide.md`

**Want to understand theory?**
→ Read `docs/paper_draft.md` Section 3

**Found a bug?**
→ Check `docs/critical_analysis.md` first

**Want to contribute?**
→ See `openpi/CONTRIBUTING.md`

---

## 📧 Contact

- **Issues**: https://github.com/yourusername/dynanfe/issues
- **Email**: your.email@example.com

---

## 📄 License

MIT License - Free to use, modify, and distribute.

---

**Print this card and keep it handy! 📌**
