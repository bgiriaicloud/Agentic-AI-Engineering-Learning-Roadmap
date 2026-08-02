# Stage 10: Cloud & AI Infrastructure — Study Guide & Notebook

This module covers the systems engineering, cloud architecture, and infrastructure management required to scale AI workloads securely in production.

---

## 📅 Study Checklist
- [ ] Choose appropriate cloud GPU/TPU hardware configurations based on model requirements.
- [ ] Configure GPU node pools on Google Kubernetes Engine (GKE) or AWS EKS.
- [ ] Write Terraform templates to provision VPCs, Private Endpoints, and Vector Databases.
- [ ] Set up Workload Identity to connect cloud services without using static keys.
- [ ] Configure global load balancers with SSL termination and API rate limiting.
- [ ] Implement autoscaling for GPU nodes using queue depth metrics.

---

## 🎛️ GPU & TPU Hardware Selection Guide

Different workloads require different hardware configurations to balance performance and cost:

| Accelerator | VRAM | Intended Workload | Key Strengths |
| :--- | :--- | :--- | :--- |
| **NVIDIA H100** | 80GB HBM3 | Large LLM Pre-training / High-throughput serving. | Transformer Engine, high FP8 computation, high interconnect bandwidth (NVLink). |
| **NVIDIA A100** | 40GB / 80GB | Fine-tuning, medium LLM serving. | Strong performance, widely available across cloud providers. |
| **NVIDIA L4** | 24GB GDDR6 | inference serving, QLoRA training. | Highly cost-effective, low power draw, ideal for smaller models (e.g., 8B). |
| **Google TPU v5e** | 16GB HBM2 | Medium pre-training, fine-tuning, high-throughput inference. | Cost-effective, optimized for JAX, PyTorch, and TensorFlow in Google Cloud. |

---

## ☸️ Kubernetes GPU Workload Manifest Example

Running AI workloads (like a vLLM inference server) on Kubernetes requires scheduling models on nodes with GPU resources.

Below is a Kubernetes Deployment manifest demonstrating how to request GPU resources:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-llama-serving
  namespace: ai-workloads
spec:
  replicas: 2
  selector:
    matchLabels:
      app: vllm-llama
  template:
    metadata:
      labels:
        app: vllm-llama
    spec:
      containers:
      - name: vllm-container
        image: vllm/vllm-openai:latest
        args: ["--model", "meta-llama/Meta-Llama-3-8B-Instruct", "--port", "8000"]
        ports:
        - containerPort: 8000
        env:
        - name: HUGGING_FACE_HUB_TOKEN
          valueFrom:
            secretKeyRef:
              name: hf-secret
              key: token
        resources:
          limits:
            cpu: "4"
            memory: 16Gi
            nvidia.com/gpu: "1" # Request exactly 1 GPU node
          requests:
            cpu: "2"
            memory: 8Gi
            nvidia.com/gpu: "1"
      nodeSelector:
        cloud.google.com/gke-gpu: "true" # Force scheduling on a GPU-enabled node pool
      tolerations:
      - key: "nvidia.com/gpu"
        operator: "Exists"
        effect: "NoSchedule"
```

---

## 🛡️ Cloud Security: Network Isolation & Workload Identity

1.  **VPC and Private Endpoints:** In enterprise systems, vector databases and model servers must run within a private subnet. External internet access is blocked. Applications connect to databases using Private Links or Private Service Connect.
2.  **Workload Identity:** Avoid checking service account JSON keys into code or mounting them inside containers. **Workload Identity** maps Kubernetes service accounts directly to cloud IAM roles. The cloud provider validates the pod identity automatically, granting permissions dynamically.

---

## ❓ Common Interview Q&As

#### Q1: How do you handle autoscaling for GPU node pools?
**Answer:** Scaling GPU pools based on standard CPU or Memory metrics is ineffective because inference workloads can saturate VRAM while CPU usage remains low. 
Instead, scale based on:
1.  **Concurrency Request Queues:** Monitor the number of active requests queued at the API Gateway or model server (e.g., vLLM metrics).
2.  **Prometheus Adapter:** Configure a Kubernetes Horizontal Pod Autoscaler (HPA) using custom Prometheus metrics (e.g., `vllm:num_requests_waiting`). When waiting requests exceed a set threshold, the HPA triggers the cluster to spin up new GPU nodes.

#### Q2: What is NVLink, and why is it important for distributed LLM training and inference?
**Answer:** NVLink is NVIDIA's high-speed GPU-to-GPU interconnect technology. Standard PCIe buses are slow, creating transfer bottlenecks when splitting large models across multiple GPUs. NVLink provides up to 900 GB/s of bidirectional bandwidth, enabling multiple GPUs to function as a single accelerator. This is essential for models that exceed the VRAM of a single GPU, where layers must be split across accelerators (Tensor Parallelism).
