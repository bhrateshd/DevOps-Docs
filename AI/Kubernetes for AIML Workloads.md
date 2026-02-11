## Kubernetes for AI/ML Workloads

### Key Points

- **GPUs are expensive**: Schedule them efficiently.
- **Models need autoscaling**: Traffic and load are bursty.
- **Inference ≠ training**: Different patterns and requirements.
- **Kubernetes as control plane**: Use Kubernetes as the standard platform for AI/ML workloads.

### Types of AI/ML Workloads on Kubernetes

- **LLM inference (serving)**
  - **Use cases**: Chatbots, APIs, real-time requests.
  - **Example models**: LLaMA, Mistral, GPT-like models.
  - **Common Kubernetes resources**:
    - `Deployment`
    - `Service`
    - `HorizontalPodAutoscaler (HPA)`
    - GPU scheduling (via device plugins and resource limits).

- **ML training jobs**
  - **Use cases**: Model training, fine-tuning, batch workloads.
  - **Common Kubernetes resources**:
    - `Job`
    - `CronJob`
    - Ecosystem tools such as Kubeflow and Volcano.

- **Batch AI jobs**
  - **Use cases**: Embeddings, data preprocessing, offline inference.

### Kubernetes Architecture for LLMs

- **Core components**
  - GPU nodes (e.g., NVIDIA GPU worker nodes)
  - NVIDIA Device Plugin (to advertise GPU resources to Kubernetes)
  - Container runtime (e.g., `containerd`)
  - Model container (LLM inference server image)
  - Persistent volumes (for model weights and checkpoints)
  - `Service` / Ingress (to expose the model endpoint)

> Kubernetes doesn’t understand GPUs by default.  
> You teach Kubernetes about GPUs using the NVIDIA Device Plugin.

### Hands-On: Run an LLM Inference Pod on Kubernetes

> This demo assumes a cluster with GPU-enabled nodes (cloud GPU cluster or local cluster with GPU passthrough).

#### Step 1: Install the NVIDIA Device Plugin

```bash
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.14.0/nvidia-device-plugin.yml
```

Verify that GPUs are visible:

```bash
kubectl describe nodes | grep nvidia
```

#### Step 2: Deploy an LLM Inference Deployment

Example using HuggingFace `text-generation-inference`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: llm-inference
spec:
  replicas: 1
  selector:
    matchLabels:
      app: llm
  template:
    metadata:
      labels:
        app: llm
    spec:
      containers:
      - name: llm
        image: ghcr.io/huggingface/text-generation-inference:latest
        resources:
          limits:
            nvidia.com/gpu: 1
        env:
        - name: MODEL_ID
          value: mistralai/Mistral-7B-Instruct
        ports:
        - containerPort: 3000
```

Apply the deployment:

```bash
kubectl apply -f llm-deployment.yaml
```

#### Step 3: Expose the Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: llm-service
spec:
  selector:
    app: llm
  ports:
    - port: 80
      targetPort: 3000
```

Apply the service and test:

```bash
kubectl apply -f llm-service.yaml
kubectl port-forward svc/llm-service 8080:80
```

### Scaling LLMs on Kubernetes

- **Key ideas**
  - LLMs typically scale based on GPU utilization and latency, not just CPU.
  - One pod is usually one model replica (or one shard in more advanced setups).
  - GPU sharing is limited; more replicas generally mean higher throughput.

#### Example Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: llm-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: llm-inference
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### ML Training Jobs on Kubernetes

- **Training is different from serving**:
  - Training is typically batch, long-running, and may be distributed.
  - Serving focuses on low-latency, online inference.

#### Example `Job` for Training

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: ml-training
spec:
  template:
    spec:
      containers:
      - name: trainer
        image: pytorch/pytorch:latest
        command: ["python", "train.py"]
        resources:
          limits:
            nvidia.com/gpu: 1
      restartPolicy: Never
```

- **Advanced platforms and tools**
  - Kubeflow (ML pipelines and training operators)
  - Ray (distributed training and serving)
  - Volcano Scheduler (batch scheduling and gang scheduling)
  - Argo Workflows (orchestration for ML pipelines)

### Production Best Practices

- **Use node selectors and taints for GPU nodes**
  - Dedicate GPU nodes for AI/ML workloads and keep them isolated from generic workloads.
- **Store models on PVCs or object storage**
  - Use persistent volumes or object stores (e.g., S3, GCS) for large model weights.
- **Pre-load models to reduce cold starts**
  - Warm up pods by loading models during startup or via init containers.
- **Use PodDisruptionBudgets (PDBs)**
  - Protect critical inference workloads during node maintenance or disruptions.
- **Monitor GPU utilization**
  - Use DCGM + Prometheus + Grafana to monitor GPU metrics and rightsizing.
- **Avoid running AI workloads on shared system nodes**
  - Keep system components and AI workloads separated for reliability and security.

### Wrap-Up

- **Kubernetes can run LLMs and other AI/ML workloads effectively.**
- **GPUs become first-class citizens through device plugins and resource requests.**
- **Scaling and cost-efficiency are driven by autoscaling, right-sizing, and good scheduling.**
- **The same Kubernetes platform can serve both DevOps and AI/ML teams, making your cluster an AI platform, not just a container platform.**


