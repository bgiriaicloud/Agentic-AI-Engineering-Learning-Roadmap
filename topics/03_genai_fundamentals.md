# Stage 3: Generative AI Fundamentals — Study Guide & Notebook

This module covers the core concepts of Generative AI, prompt engineering, model architectures, and structured tool interactions.

---

## 📅 Study Checklist
- [ ] Differentiate between decoder-only, encoder-decoder, and encoder-only models.
- [ ] Explain how a base model is converted to an instruct model using SFT and RLHF/DPO.
- [ ] Write system prompts that successfully constrain LLM behavior.
- [ ] Implement structured output from an LLM using Pydantic.
- [ ] Write an application that parses tool calling responses from an LLM.
- [ ] Compare the trade-offs of commercial APIs (e.g., Claude, Gemini) and open-weights models (e.g., Llama 3).

---

## 🏗️ LLM Architectures

Different tasks require different architectural designs:

| Architecture | Primary Focus | Examples | Common Use Cases |
| :--- | :--- | :--- | :--- |
| **Encoder-Only** | Bidirectional context representation. | BERT, RoBERTa | Classification, NER, Embedding generation. |
| **Decoder-Only** | Autoregressive token prediction. | GPT-4, Llama, Claude | Creative writing, code generation, reasoning. |
| **Encoder-Decoder** | Mapping inputs to output sequences. | T5, BART | Translation, text summarization. |

---

## 🛠️ Instruction Tuning & RLHF

Base LLMs are next-token predictors trained on raw internet text. They do not follow commands; instead, they continue the text.
To make them useful assistants, models undergo:
1.  **Supervised Fine-Tuning (SFT):** The model is trained on high-quality, formatted instruction-response pairs (e.g., Prompt: "Summarize this...", Response: "Here is...").
2.  **Reinforcement Learning from Human Feedback (RLHF) / Direct Preference Optimization (DPO):** The model is fine-tuned based on preference data (which outputs humans preferred) to align the model for safety, helpfulness, and style.

---

## 📦 Enforcing Structured Outputs with Pydantic

In application development, raw text outputs from LLMs are difficult to parse programmatically. Using structured output libraries ensures that the model returns data conforming to a validated schema.

Here is a practical example using `Instructor` and `Pydantic` to parse contact info from text:

```python
from typing import List
from pydantic import BaseModel, Field
import instructor
from google import genai
from google.genai import types

# 1. Define the target schema
class ContactInfo(BaseModel):
    name: str = Field(description="The full name of the person.")
    email: str = Field(description="The email address of the person.")
    phone: str | None = Field(default=None, description="The phone number, if available.")
    skills: List[str] = Field(description="List of skills mentioned in the profile.")

# Example text input
raw_profile = """
Meet Biswanath Giri. You can reach him at biswanath@example.com or call him at +1-555-0199.
He has extensive experience in Python, Cloud Architecture, and Agentic AI development.
"""

def extract_structured_data(text: str):
    # Initialize the standard Google GenAI client
    client = genai.Client()
    
    # We define the schema to pass to the model configuration
    # Note: Modern model APIs allow passing Pydantic schemas directly as response schemas
    response = client.models.generate_content(
        model='gemini-2.5-flash',
        contents=f"Extract contact info from the profile: {text}",
        config=types.GenerateContentConfig(
            response_mime_type="application/json",
            response_schema=ContactInfo,
            temperature=0.0 # Low temperature ensures deterministic extraction
        ),
    )
    
    # Validate and load the JSON string directly into the Pydantic model
    validated_data = ContactInfo.model_validate_json(response.text)
    return validated_data

if __name__ == "__main__":
    # Ensure you have your GEMINI_API_KEY environment variable set
    # result = extract_structured_data(raw_profile)
    # print(result.model_dump_json(indent=2))
    print("Code defined. Run with active API key to extract.")
```

---

## 🛠️ How Tool Calling Works under the Hood

Tool calling (or Function Calling) does not mean the LLM runs your local functions. Rather, it is a structured communication loop:

```
[ Application ] ─── (1) Sends Prompts + Tool JSON Schemas ───> [ LLM ]
       ▲                                                          │
       │                                                    (2) Decides to call Tool
       │                                                    Returns arguments in JSON
       │                                                          │
       └───── (3) Runs Local Code ─── (4) Sends Tool Output ──────┘
```

1.  The client passes a list of available tools, defined as JSON schemas (functions with parameters and descriptions), along with the prompt.
2.  The LLM reads the prompt, determines which tool to call, and returns a JSON payload containing the function name and structured arguments.
3.  The client intercepts this response, runs the corresponding function locally using the arguments provided, and gets the result.
4.  The client sends the tool result back to the LLM.
5.  The LLM summarizes the tool output and returns a final response to the user.

---

## ❓ Common Interview Q&As

#### Q1: What is the difference between temperature and top-p sampling?
**Answer:** Both control token selection diversity:
- **Temperature:** Rescales the raw logits output by the model. A high temperature (e.g., 1.2) flattens the probability distribution, making less likely tokens more common (higher creativity). A low temperature (e.g., 0.1) sharpens the distribution, forcing the model to select only the most likely tokens.
- **Top-p (Nucleus Sampling):** Sets a cumulative probability threshold. The model only considers the smallest pool of tokens whose combined probabilities sum to $\ge p$ (e.g., $p=0.9$). This drops highly unlikely tokens entirely, preventing nonsensical words.

#### Q2: How does a Vision-Language Model (VLM) process images alongside text?
**Answer:** A VLM typically uses a pre-trained **Vision Encoder** (like a ViT - Vision Transformer) to process images. The image is split into small patches, which are projected into visual token embeddings. These visual tokens are aligned (via a projection layer) to match the embedding dimension of the language model. They are then prepended to the text tokens, allowing the text Transformer to attend to both visual and text tokens.
