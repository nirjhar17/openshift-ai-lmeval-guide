# OpenShift AI LM-Eval Guide

A comprehensive, hands-on guide to evaluating Large Language Models (LLMs) using **Red Hat OpenShift AI 3.0** and the **TrustyAI Operator**.

## 📖 Read the Guide

**[Complete LM-Eval Guide →](blog/lmeval-complete-guide.md)**

## What You'll Learn

- How LM-Eval works on OpenShift AI
- Testing different model types (HuggingFace, Local, OpenAI)
- Pre-built vs Custom evaluations using Unitxt
- Understanding Cards, Templates, and Tasks
- Real benchmark results and comparisons

## Model Types Covered

| Model Type | Description |
|------------|-------------|
| `hf` | HuggingFace models (downloaded at runtime) |
| `local-completions` | Models deployed on OpenShift (vLLM/KServe) |
| `local-chat-completions` | Chat models on OpenShift |
| `openai-completions` | OpenAI API models |
| `openai-chat-completions` | OpenAI chat models (GPT-3.5/4) |

## Results Summary

| Model | Task | Score |
|-------|------|-------|
| flan-t5-small (HF) | wnli | 56% |
| Qwen3-0.6B (local) | arc_easy | 60% |
| gpt-3.5-turbo (OpenAI) | gsm8k | 80% |
| Qwen3-0.6B (local) | custom unitxt | 50% / 67% F1 |

## Repository Structure

```
├── blog/                      # Guide and documentation
├── phase1-setup/              # Namespace setup
├── phase3-hf-model/           # HuggingFace model examples
├── phase4-local-completions/  # Local model examples
├── phase5-local-chat/         # Chat completion examples
├── phase6-openai-completions/ # OpenAI examples
├── phase7-openai-chat/        # OpenAI chat examples
├── phase8-task-types/         # Different task configurations
├── phase9-custom-unitxt/      # Custom evaluation examples
└── phase10-pvc-storage/       # PVC storage examples
```

## Prerequisites

- Red Hat OpenShift AI 3.0
- TrustyAI Operator enabled
- Access to deploy LMEvalJob resources

## License

This guide is provided for educational purposes.

---

*Created while exploring LM-Eval on OpenShift AI 3.0*
