# Stage 6: Model Training & Adaptation — Study Guide & Notebook

This module covers the systems engineering, optimization mechanics, and VRAM math required to fine-tune, quantize, and serve open-weights LLMs.

---

## 📅 Study Checklist
- [ ] Calculate the exact VRAM required to load and train a given model parameter size.
- [ ] Explain the mathematical mechanics of Low-Rank Adaptation (LoRA).
- [ ] Configure and launch a QLoRA fine-tuning run using Hugging Face PEFT and Unsloth.
- [ ] Differentiate between GGUF, GPTQ, and AWQ quantization formats.
- [ ] Run post-training quantization on a model and evaluate model degradation.
- [ ] Set up a knowledge distillation pipeline.

---

## 🔢 VRAM Mathematics: Sizing Compute Requirements

Estimating VRAM requirements is critical before reserving cloud GPU resources.

### 1. VRAM Required for Inference
To load model weights, memory usage depends on parameter size $P$ (in billions) and precision bytes $B$ (FP32 = 4 bytes, FP16/BF16 = 2 bytes, INT8 = 1 byte, INT4 = 0.5 bytes):
$$\text{Weight Memory} = P \times B\text{ GB}$$
*   **KV Cache overhead:** Storing historical context keys and values during generation requires additional VRAM.
    $$\text{KV Cache Memory} = 2 \times \text{Batch Size} \times \text{Seq Len} \times \text{Layers} \times \text{Heads} \times \text{Dim} \times \text{Bytes per Param}$$
*   **Rule of Thumb:** A 7B parameter model in FP16 requires $7 \times 2 = 14\text{ GB}$ of VRAM just to load weights. Add $\approx 4\text{ GB}$ for execution overhead and KV Cache; it requires a minimum of 24GB of VRAM (e.g., a single RTX 3090/4090 or L4 GPU).

### 2. VRAM Required for Fine-Tuning
Training requires significantly more VRAM than inference because it must store optimizer states, gradients, and activation states:
*   **Full training (AdamW optimizer):** Requires $\approx 16$ to $18$ bytes per parameter.
    *   Weights: 4 bytes (stored in FP32 for weight updates).
    *   Gradients: 4 bytes.
    *   Optimizer States (AdamW): 8 bytes (stores momentum and variance tracking).
    *   **Total:** $16 \times 7 = 112\text{ GB}$ of VRAM for a 7B model, plus memory for activation states.
*   **LoRA training:** Freezes base weights (in FP16 = 2 bytes/param) and updates only small adapter matrices, keeping optimizer states and gradients limited to the small adapter weights ($< 1\%$ of model parameters).

---

## 📉 LoRA & QLoRA Mechanics

Instead of updating all weights in a weight matrix $W_0 \in \mathbb{R}^{d \times k}$, **Low-Rank Adaptation (LoRA)** factorizes weight updates $\Delta W$ into two low-rank matrices $A \in \mathbb{R}^{d \times r}$ and $B \in \mathbb{R}^{r \times k}$, where the rank $r \ll d, k$.

```
                 Weight Update (Merged)
                 ┌──────────────────┐
                 │    W = W0 + ΔW   │
                 └──────────────────┘
                         ▲
                        / \
                       /   \
        Original Weights   Weight Adapters (LoRA)
         (Frozen)             (Trainable)
        ┌──────────────┐     ┌──────────────┐
        │  W0 (d x k)  │     │   A (d x r)  │  (r dimensions)
        └──────────────┘     └──────────────┘
                             │   B (r x k)  │
                             └──────────────┘
```

*   **Mathematical Formula:**
    $$h = W_0 x + \Delta W x = W_0 x + \frac{\alpha}{r} (B A) x$$
    Where $\alpha$ is a scaling hyperparameter.
*   **QLoRA (Quantized LoRA):** Quantizes the base weights $W_0$ down to **4-bit NormalFloat (NF4)** precision, saving memory. During forward and backward passes, the base weights are temporarily dequantized to FP16 to calculate gradients for the trainable FP16 LoRA adapters.

---

## 🗜️ Quantization Formats & Serving Profiles

Quantization reduces weights to lower bit representations post-training:

| Format | Primary Platform | Use Case | Features |
| :--- | :--- | :--- | :--- |
| **GGUF** | CPU, Mac, local systems. | Local personal execution. | Designed to load and run on host RAM alongside CPU/GPU offloading. |
| **GPTQ** | GPUs. | Production GPU clusters. | Calibrates weight layers one-by-one; fast execution on standard Linux/GPU runtimes. |
| **AWQ** | GPUs. | Enterprise serving gateways. | Activation-aware Quantization. Identifies and protects critical weights, minimizing accuracy loss. |

---

## ❓ Common Interview Q&As

#### Q1: What is the difference between SFT (Supervised Fine-Tuning) and DPO (Direct Preference Optimization)?
**Answer:**
- **SFT:** Trains a model to predict the next token based on target label examples. It teaches the model style, structure, and formatting.
- **DPO:** Skips SFT's reinforcement learning steps (which require building complex reward models). It directly optimizes the model policy based on paired human feedback data (a preferred output vs an rejected output) using a binary cross-entropy loss, aligning the model with safety and style guidelines.

#### Q2: When should you fine-tune a model instead of using RAG?
**Answer:**
*   **Use RAG when:** The dataset updates frequently (e.g., news or stock prices), you need to cite source links, or data access must be controlled using user authorization levels.
*   **Use Fine-Tuning when:** You need to teach the model a specific style, language format, or custom code syntax, or when you want to minimize token usage by embedding instructions directly into the model's weights.
