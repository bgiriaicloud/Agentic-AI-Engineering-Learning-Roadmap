# Stage 2: Machine Learning & Deep Learning Basics — Study Guide & Notebook

This module covers the core concepts of Machine Learning, Deep Neural Networks, and the Transformer architecture that underpins all modern LLMs.

---

## 📅 Study Checklist
- [ ] Differentiate between supervised, unsupervised, and reinforcement learning paradigms.
- [ ] Understand forward propagation, backpropagation, and gradient descent optimization.
- [ ] Implement a basic neural network layer using PyTorch.
- [ ] Explain the components of a Transformer layer: self-attention, projection, layer norm, residual connections.
- [ ] Code a basic text tokenizer (BPE) from scratch in Python.
- [ ] Explain how text is converted to embeddings and how cosine similarity is used to compare them.
- [ ] Describe the differences between FP32, FP16, and BF16 numerical formats and their VRAM impacts.

---

## 🏗️ Neural Networks 101

A Neural Network is a collection of stacked layers of parameterized mathematical operations.
1.  **Forward Propagation:** Passes input $X$ through layers to calculate output prediction $\hat{y}$.
    $$\hat{y} = \sigma(W \cdot X + b)$$
    Where $W$ represents weights, $b$ is the bias, and $\sigma$ is a non-linear activation function (like ReLU or GeLU) that enables learning complex non-linear structures.
2.  **Loss Function:** Computes error $L$ between the prediction $\hat{y}$ and true label $y$ (e.g., Mean Squared Error or Cross-Entropy Loss).
3.  **Backpropagation:** Calculates the gradient of the loss with respect to every weight $\frac{\partial L}{\partial W}$ using the Chain Rule of calculus.
4.  **Optimizer (Gradient Descent):** Updates the weights slightly in the opposite direction of the gradients to minimize loss.
    $$W_{new} = W_{old} - \eta \frac{\partial L}{\partial W}$$
    Where $\eta$ is the learning rate.

---

## 🔀 The Transformer & Attention Mechanics

Introduced in the paper *"Attention Is All You Need"* (2017), the Transformer replaces recurrent layers (LSTMs) with self-attention, allowing models to process all tokens in a sequence concurrently.

### 1. Tokenization
Before passing text into a model, it must be sliced into numerical IDs. Modern tokenizers use **Byte-Pair Encoding (BPE)**:
1.  Start with a vocabulary of individual characters.
2.  Count the frequency of adjacent token pairs in a training corpus.
3.  Merge the most frequent pair into a single token.
4.  Repeat until the target vocabulary size (e.g., 32,000 or 128,000) is reached.

### 2. Self-Attention Mechanics
Self-attention computes how much every word in a sequence should pay attention to every other word.
*   Given input embeddings $X$, we multiply them by weight matrices to create three representations:
    *   **Query (Q):** What the word is looking for.
    *   **Key (K):** What characteristics the word offers.
    *   **Value (V):** The actual informational content of the word.
*   **Formula:**
    $$\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V$$
    *   $QK^T$ calculates raw similarity scores between all queries and keys.
    *   Dividing by $\sqrt{d_k}$ (scaling factor) prevents gradients from exploding during training.
    *   `softmax` converts similarity scores into a probability distribution summing to 1.
    *   Multiplying by $V$ computes a weighted sum of values based on attention weights.

---

## 🖥️ PyTorch Implementation: Simple Transformer Attention

Here is a simplified PyTorch implementation of Self-Attention to understand the math in code.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class SimpleSelfAttention(nn.Module):
    def __init__(self, embed_dim: int):
        super().__init__()
        self.embed_dim = embed_dim
        
        # Projection matrices
        self.q_proj = nn.Linear(embed_dim, embed_dim, bias=False)
        self.k_proj = nn.Linear(embed_dim, embed_dim, bias=False)
        self.v_proj = nn.Linear(embed_dim, embed_dim, bias=False)
        
        # Scaling factor
        self.scale = torch.sqrt(torch.tensor(embed_dim, dtype=torch.float32))

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        # x shape: [batch_size, sequence_length, embed_dim]
        batch_size, seq_len, embed_dim = x.shape
        
        # Project inputs to Q, K, V
        Q = self.q_proj(x) # [batch_size, seq_len, embed_dim]
        K = self.k_proj(x) # [batch_size, seq_len, embed_dim]
        V = self.v_proj(x) # [batch_size, seq_len, embed_dim]
        
        # Compute dot product attention scores
        # We transpose K to align matrices for dot products
        scores = torch.matmul(Q, K.transpose(-2, -1)) / self.scale # [batch_size, seq_len, seq_len]
        
        # Apply softmax to get attention weights
        attention_weights = F.softmax(scores, dim=-1) # [batch_size, seq_len, seq_len]
        
        # Multiply weights by Values
        out = torch.matmul(attention_weights, V) # [batch_size, seq_len, embed_dim]
        
        return out

if __name__ == "__main__":
    # Test case: Batch size of 1, Sequence length of 3 tokens, Embedding size of 8 dimensions
    dummy_input = torch.randn(1, 3, 8)
    attention_layer = SimpleSelfAttention(embed_dim=8)
    output = attention_layer(dummy_input)
    print("Input shape:", dummy_input.shape)
    print("Output shape:", output.shape)
```

---

## ⚡ Hardware & Numeric Formats (BF16 vs FP16 vs FP32)

Running deep learning models requires balancing precision and memory usage.

```
FP32 (32-bit Single Precision):
[ 1 Sign bit ] [   8 Exponent bits   ] [         23 Fraction/Mantissa bits         ]
FP16 (16-bit Half Precision):
[ 1 Sign bit ] [  5 Exponent bits  ] [      10 Fraction/Mantissa bits      ]
BF16 (16-bit Brain Float):
[ 1 Sign bit ] [   8 Exponent bits   ] [   7 Fraction/Mantissa bits   ]
```

*   **FP32 (Float32):** Standard single precision. High accuracy, but requires 4 bytes of memory per parameter. Used during training when numerical stability is critical.
*   **FP16 (Float16):** 16-bit float. Requires 2 bytes of memory. Susceptible to numerical underflow or overflow because of its narrow range of exponent values (5 bits).
*   **BF16 (Bfloat16):** Developed by Google. Matches the range of FP32 (8 exponent bits) but has lower precision (7 fraction bits). Highly stable for training and inference, supported on modern GPUs (Ampere architectures and newer) and TPUs.

---

## ❓ Common Interview Q&As

#### Q1: Why are activation functions needed in neural networks? What happens if you stack linear layers without them?
**Answer:** Activation functions introduce non-linearity. Stacking multiple linear layers without activation functions results in a single linear transformation, as:
$$W_2 \cdot (W_1 \cdot X + b_1) + b_2 = (W_2 \cdot W_1) \cdot X + (W_2 \cdot b_1 + b_2) = W_{combined} \cdot X + b_{combined}$$
Without non-linear activations, the network would only be able to learn simple linear boundaries, rendering deep network depth useless.

#### Q2: What is the purpose of Positional Encoding in the Transformer?
**Answer:** Because the Transformer processes all input tokens concurrently via attention, it has no inherent sense of word order (sequence order). To solve this, **Positional Encodings** are added to the input embeddings before they enter the model. These encodings use sine and cosine functions of varying frequencies to inject spatial information directly into the representation.
