# 4. Low-Level Design (LLD)

> [!NOTE]
> This document is the **engineering blueprint**. It contains the exact configurations, code snippets, and technical decisions that a developer needs to implement the architecture described in the HLD. If the HLD is the "what," the LLD is the "exactly how."

---

## 4.1 CI/CD Pipeline Architecture

### Pipeline Stages (Detailed)

![CI/CD Pipeline Architecture](images/cicd_pipeline.png)

### CodeBuild `buildspec.yml`

The CI/CD pipeline is orchestrated via AWS CodeBuild. The build specification defines strict phases: dependency installation, pre-build security/linting checks, Docker image compilation, and post-build model registry integration. This ensures that only fully tested and validated containers reach the deployment stage.

---

## 4.2 Serving Stack Deep-Dive

### Inference Framework Comparison

| Feature | **vLLM** ⭐ (Recommended) | **TGI** (Hugging Face) | **Triton** (NVIDIA) |
|---------|--------------------------|------------------------|---------------------|
| **Throughput** | Highest (PagedAttention) | High (FlashAttention) | High (custom backends) |
| **Continuous Batching** | ✅ Native | ✅ Native | ✅ With custom config |
| **Streaming** | ✅ SSE support | ✅ SSE support | ⚠️ gRPC streaming only |
| **Quantization** | ✅ AWQ, GPTQ, FP8 | ✅ AWQ, GPTQ | ✅ TensorRT-LLM |
| **SageMaker Integration** | ✅ Custom container | ✅ HF DLC available | ✅ NVIDIA DLC available |
| **Complexity to Operate** | Low | Low | High |
| **Best For** | SLMs with high throughput needs | Quick HuggingFace model deploys | Multi-model, multi-framework |
| **Community & Docs** | Excellent, fast-growing | Excellent (HuggingFace) | Good but enterprise-focused |

> [!TIP]
> **Why vLLM for this project?** Three reasons:
> 1. **PagedAttention** — Manages GPU memory like an OS manages RAM pages, reducing memory waste by up to 90%. For SLMs on a single GPU, this means you can serve more concurrent requests.
> 2. **Continuous Batching** — Instead of waiting for a full batch, vLLM dynamically adds/removes requests from the batch as they arrive/complete. This keeps the GPU busy at all times.
> 3. **Simple API** — OpenAI-compatible API out of the box, so your client's applications can switch from OpenAI to your SLM by just changing the base URL.

### Dockerfile for Inference Container

The inference container utilizes a multi-stage Docker build to ensure a minimal and secure production footprint. It operates with a non-root user, limits exposed attack surfaces, and encapsulates the vLLM server optimized specifically for our GPU instance architecture.

### `requirements-inference.txt`

The python dependencies are strictly pinned to exact versions, including the vLLM engine, tokenizers, and observability tools, guaranteeing reproducible builds and preventing supply-chain vulnerabilities.

### Model Loading Configuration

A custom configuration module initializes the serving engine. It dynamically allocates GPU memory utilization (reserving a safety buffer for OS operations) and configures continuous batching parameters to maximize throughput without hitting Out-Of-Memory (OOM) errors.

---

## 4.3 Security Posture (Detailed)

### 4.3.1 VPC Network Architecture

![VPC Network Architecture](images/vpc_network.png)

### 4.3.2 Subnet Allocation

| Subnet | CIDR | AZ | Purpose | Internet Access |
|--------|------|----|---------|-----------------|
| `pub-az1` | `10.0.1.0/24` | us-east-1a | NAT Gateway, ALB | Direct (IGW) |
| `pub-az2` | `10.0.2.0/24` | us-east-1b | NAT Gateway, ALB | Direct (IGW) |
| `priv-az1` | `10.0.10.0/24` | us-east-1a | SageMaker, ECS | Outbound only (NAT) |
| `priv-az2` | `10.0.20.0/24` | us-east-1b | SageMaker, ECS | Outbound only (NAT) |
| `vpce` | `10.0.100.0/24` | us-east-1a | VPC Endpoints | None |

> [!IMPORTANT]
> **Why two Availability Zones (AZs)?** If one AZ experiences an outage (rare but possible), the other AZ continues serving traffic. This is how we achieve the 99.9% uptime SLA from the PRD. Always deploy critical infrastructure across at least 2 AZs.

### 4.3.3 Security Groups

| Security Group | Attached To | Inbound Rules | Outbound Rules |
|---------------|-------------|---------------|----------------|
| `sg-alb` | Application Load Balancer | `443/tcp` from `0.0.0.0/0` (HTTPS only) | `8080/tcp` to `sg-sagemaker` |
| `sg-sagemaker` | SageMaker Endpoint instances | `8080/tcp` from `sg-alb` only | `443/tcp` to VPC Endpoints |
| `sg-vpce` | VPC Endpoint ENIs | `443/tcp` from `sg-sagemaker` | N/A (managed by AWS) |
| `sg-codebuild` | CodeBuild projects | None (no inbound needed) | `443/tcp` to `0.0.0.0/0` (for package downloads) |

> [!NOTE]
> **Security Group Principle:** Each group only allows the minimum traffic needed. The SageMaker instances can *only* receive traffic from the ALB (not from the internet directly), and can *only* make outbound calls to VPC Endpoints (not to arbitrary internet hosts).

### 4.3.4 IAM Roles (Least Privilege)

Every component in the architecture (SageMaker, CodeBuild, CodePipeline) is assigned a dedicated IAM role. These roles strictly adhere to the principle of least privilege, explicitly granting access only to the specific S3 buckets, ECR repositories, and KMS keys required for their function. This minimizes the blast radius of any potential compromise.

### 4.3.5 WAF Rules Configuration

The API Gateway is protected by an AWS Web Application Firewall (WAF). The configuration includes aggressive rate limiting to prevent GPU exhaustion, blocks known malicious payloads (OWASP Top 10), restricts request body sizes to prevent abuse, and can enforce geo-blocking compliance requirements.

---

## 4.4 Observability

### 4.4.1 Key Metrics to Track

| Category | Metric | Source | Alarm Threshold | Why It Matters |
|----------|--------|--------|-----------------|----------------|
| **Latency** | `InferenceLatency_P50` | vLLM + CloudWatch | > 200ms | Indicates model or GPU is struggling |
| **Latency** | `InferenceLatency_P99` | vLLM + CloudWatch | > 500ms | Tail latency affects worst-case UX |
| **Throughput** | `TokensPerSecond` | vLLM custom metric | < 30 tokens/s | Model is underperforming |
| **Throughput** | `RequestsPerSecond` | ALB metrics | > 80% of max capacity | Time to scale out |
| **GPU** | `GPUUtilization` | SageMaker built-in | > 85% sustained for 5 min | GPU is saturated; scale out |
| **GPU** | `GPUMemoryUtilization` | SageMaker built-in | > 90% | Risk of OOM; check for memory leaks |
| **Errors** | `5xxErrorRate` | ALB + API Gateway | > 1% | Something is broken |
| **Errors** | `ModelLatencyAlarm` | SageMaker | Invocation errors > 5/min | Model container is crashing |
| **Availability** | `HealthyHostCount` | ALB Target Group | < 2 | Below HA threshold |
| **Cost** | `EstimatedCharges` | CloudWatch Billing | > $X/day | Cost spike detection |

### 4.4.2 Custom Metrics Emission (Python)

Beyond standard AWS metrics, the serving code includes custom telemetry to track SLM-specific KPIs. By emitting fine-grained metrics like 'Tokens Per Second' and 'Inference Latency' directly to CloudWatch, the system enables highly accurate scaling policies and real-time performance monitoring.

### 4.4.3 CloudWatch Dashboard Definition

A unified CloudWatch Dashboard provides a single pane of glass for monitoring the model's health. It aggregates P50/P99 latencies, GPU saturation, HTTP 5xx error rates, and estimated daily AWS costs, giving operational teams instant visibility.

### 4.4.4 Alarm Configuration

Automated CloudWatch Alarms act as a 24/7 on-call system. Thresholds are set for high tail-latencies, error rate spikes, and sustained GPU saturation. When breached, these alarms trigger auto-scaling events or immediately page the engineering team via SNS integrations.

---

## 4.5 Auto-Scaling Configuration

The SageMaker endpoint utilizes a Target Tracking Auto-Scaling policy. It continuously monitors GPU utilization, scaling out to handle traffic spikes and scaling in during off-peak hours to minimize cloud spend, while always maintaining a minimum baseline capacity for high availability.

---

## 4.6 Terraform Module Structure

The entire infrastructure is defined as code using Terraform. It is heavily modularized (separating VPC networking, security, compute, and CI/CD into independent components). This modular approach allows for parallel development, safe state management, and easy replication across staging and production environments.

> [!TIP]
> **Why modular Terraform?** Each module can be developed, tested, and deployed independently. If you need to update a WAF rule, you don't risk accidentally modifying the VPC. This also enables different team members to work on different modules in parallel without merge conflicts.
