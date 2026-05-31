# 6. MLOps & AWS Glossary

> [!NOTE]
> This glossary defines every specialized term, AWS service, and MLOps concept used across the previous 5 artifacts. Terms are grouped by category and include brief explanations of **why** each concept matters for this specific SLM deployment project.

---

## AWS Services

| Term | Full Name | Definition | Why It Matters Here |
|------|-----------|------------|---------------------|
| **ALB** | Application Load Balancer | A Layer 7 (HTTP/HTTPS) load balancer that distributes incoming traffic across multiple targets (e.g., SageMaker instances). It inspects HTTP headers to make routing decisions. | Distributes inference requests across multiple GPU instances. Performs health checks and removes unhealthy instances from rotation. |
| **API Gateway** | Amazon API Gateway | A fully managed service that creates, publishes, and manages APIs. Acts as the "front door" for your API, handling authentication, rate limiting, and request routing. | Provides a managed, secure entry point for the SLM API. Handles API keys, throttling, and request validation without custom code. |
| **CloudFormation** | AWS CloudFormation | Infrastructure-as-Code service that lets you define AWS resources in JSON/YAML templates and deploy them as "stacks." | Used internally by SageMaker for blue/green deployments. Terraform (our primary IaC tool) can interact with it. |
| **CloudTrail** | AWS CloudTrail | Records all API calls made in your AWS account — who did what, when, and from where. Think of it as a security camera for your AWS account. | Provides the audit trail required for compliance. If a model is deleted or a security group is modified, CloudTrail tells you who did it. |
| **CloudWatch** | Amazon CloudWatch | AWS's monitoring and observability service. Collects metrics, logs, and events from AWS resources and custom applications. | Central hub for all monitoring: inference latency, GPU utilization, error rates, and custom token-level metrics. Powers automated alarms. |
| **CodeBuild** | AWS CodeBuild | Fully managed build service that compiles code, runs tests, and produces deployable artifacts (e.g., Docker images). | Executes each CI/CD pipeline stage: linting, testing, Docker image building, and security scanning. No Jenkins servers to maintain. |
| **CodeCommit** | AWS CodeCommit | AWS's managed Git repository service. Like a private GitHub but within AWS. | Optional source code host. Can be swapped for GitHub — CodePipeline supports both as source providers. |
| **CodePipeline** | AWS CodePipeline | Continuous delivery service that orchestrates build, test, and deploy stages. Visualizes and automates the release pipeline. | Orchestrates the entire CI/CD flow: source → build → test → scan → register → staging → approval → production. |
| **DynamoDB** | Amazon DynamoDB | Fully managed NoSQL database with single-digit millisecond performance at any scale. | Stores model configuration metadata and deployment state. Fast reads ensure no added latency to the inference path. |
| **ECR** | Elastic Container Registry | AWS's Docker container image registry. Stores, manages, and deploys container images. Includes built-in vulnerability scanning. | Stores the Docker images containing vLLM and the model serving code. Scans images for CVEs before deployment. |
| **ECS** | Elastic Container Service | Container orchestration service that runs Docker containers on AWS. Fargate mode eliminates the need to manage servers. | Used for async/batch inference workloads and model evaluation jobs that don't need GPU. |
| **Fargate** | AWS Fargate | Serverless compute engine for containers. You define CPU/memory requirements; AWS handles the rest. | Runs ECS tasks without provisioning or managing EC2 instances. Pay only for the compute time you use. |
| **IAM** | Identity and Access Management | AWS's permission system. Controls who (identity) can do what (action) on which resources. Every AWS API call is authorized by IAM. | Each component (SageMaker, CodeBuild, CodePipeline) gets its own IAM role with minimum required permissions — the "least privilege" principle. |
| **Inspector** | Amazon Inspector | Automated vulnerability assessment service that scans container images and EC2 instances for software vulnerabilities and network exposure. | Performs deep CVE scanning on every container image pushed to ECR. Catches vulnerabilities that ECR's basic scan might miss. |
| **KMS** | Key Management Service | Managed service for creating and controlling encryption keys. You define who can use keys; AWS handles the cryptographic operations. | Encrypts model artifacts in S3, container images in ECR, and logs. Customer-managed keys give the client control over their encryption. |
| **NAT Gateway** | Network Address Translation Gateway | Allows instances in private subnets to access the internet (e.g., to download packages) while preventing the internet from initiating connections to those instances. | SageMaker instances in private subnets need outbound internet access during setup (downloading model weights, pip packages) without being directly reachable. |
| **S3** | Simple Storage Service | Object storage service with 99.999999999% (11 nines) durability. Stores any amount of data, from bytes to petabytes. | Stores model weight files (.safetensors), training artifacts, deployment logs, and Terraform state. Versioning enables model rollback. |
| **SageMaker** | Amazon SageMaker | End-to-end ML platform. In this project, we use its **Real-Time Endpoints** feature — managed infrastructure that hosts your model and serves predictions. | Primary compute service for model inference. Handles GPU provisioning, auto-scaling, health monitoring, and blue/green deployments. |
| **Secrets Manager** | AWS Secrets Manager | Securely stores and rotates secrets (API keys, database passwords, tokens). Integrates with IAM for access control. | Stores API keys, Docker Hub tokens, and any other sensitive configuration. No secrets are hardcoded in code or environment variables. |
| **SNS** | Simple Notification Service | Pub/sub messaging service. Sends notifications to email, SMS, Slack (via Lambda), or other AWS services. | Delivers alarm notifications when metrics breach thresholds. Sends deployment approval requests and incident alerts. |
| **VPC** | Virtual Private Cloud | An isolated virtual network within AWS. You control IP ranges, subnets, routing, and network gateways. Think of it as your own private data center in the cloud. | All compute resources run inside a VPC. Public subnets host the ALB; private subnets host SageMaker endpoints. This ensures the model is never directly exposed to the internet. |
| **VPC Endpoint** | VPC Endpoint (PrivateLink) | A private connection between your VPC and AWS services that doesn't traverse the public internet. Like a dedicated tunnel to an AWS service. | Allows SageMaker instances in private subnets to access S3, ECR, and CloudWatch without needing internet access. Reduces data transfer costs and improves security. |
| **WAF** | Web Application Firewall | Protects web applications from common exploits (SQL injection, XSS, DDoS). Inspects HTTP requests and blocks malicious patterns. | Sits in front of API Gateway. Blocks malicious requests, enforces rate limits per IP, and can restrict access by geography. |
| **X-Ray** | AWS X-Ray | Distributed tracing service. Visualizes how a request flows through your application, showing latency at each hop. | Traces a request from API Gateway → ALB → SageMaker, pinpointing exactly where latency is introduced. Essential for debugging slow responses. |

---

## MLOps Concepts

| Term | Definition | Why It Matters Here |
|------|------------|---------------------|
| **MLOps** | Machine Learning Operations. The practice of applying DevOps principles (CI/CD, monitoring, automation) to ML systems. Bridges the gap between training a model and running it reliably in production. | This entire project is an MLOps implementation. Without MLOps practices, model deployment is manual, error-prone, and impossible to scale. |
| **Model Registry** | A centralized repository for storing, versioning, and managing ML model artifacts. Tracks model lineage (what data and code produced each version) and approval status. | SageMaker Model Registry is the single source of truth for which model versions exist and which one is approved for production. Enables reliable rollback. |
| **Model Lineage** | The complete record of how a model was produced: training data, code version, hyperparameters, metrics. Like a "birth certificate" for a model. | If a model starts producing bad results, lineage lets you trace back to the exact training run and data that created it. |
| **Model Serving** | The process of hosting a trained model and making it available for predictions (inference) via an API. Also called "model deployment" or "model hosting." | The core purpose of this project — taking the fine-tuned SLM and serving it as a REST API. |
| **Inference** | Using a trained model to generate predictions (outputs) from new inputs. In the SLM context, inference = generating text from a prompt. | Every API request to the SLM endpoint triggers an inference. Optimizing inference speed and cost is the primary technical challenge. |
| **Fine-Tuning** | Adapting a pre-trained model to a specific domain or task by training it further on a smaller, specialized dataset. The model retains its general knowledge while gaining domain expertise. | The client has already fine-tuned the SLM. This project deploys the fine-tuned result. Understanding fine-tuning helps you understand why the model artifacts look the way they do. |
| **Model Artifact** | The output files from training: model weights (.safetensors, .bin), tokenizer files, and configuration. These are the files needed to load and run the model. | Stored in S3, versioned in Model Registry. The container downloads these at startup and loads them into GPU memory. |
| **Blue/Green Deployment** | A deployment strategy that maintains two identical environments: "blue" (current production) and "green" (new version). Traffic is switched from blue to green only after the green environment passes all health checks. | Ensures zero downtime during model updates. If the new model has issues, traffic can instantly be switched back to the old version. |
| **Canary Deployment** | A strategy where a small percentage of traffic (e.g., 10%) is routed to the new version while 90% stays on the old version. If metrics are healthy, traffic gradually shifts to 100%. | Reduces risk of a bad model update affecting all users simultaneously. If the canary shows degraded metrics, the rollout is halted and rolled back. |
| **Rollback** | Reverting a deployment to a previous known-good version. In MLOps, this means switching back to the previous model version and container image. | Critical safety net. If a new model version causes high latency or incorrect outputs, rollback restores service in minutes, not hours. |
| **Shadow Deployment** | Running a new model version alongside the current production version. Both receive the same traffic, but only the current version's responses are returned to users. The shadow version's responses are logged for comparison. | Enables safe evaluation of a new model in production conditions without any risk to users. Used before canary deployment for high-risk changes. |
| **A/B Testing** | Splitting traffic between two model versions to compare their performance on real user requests. Unlike canary (which tests reliability), A/B testing evaluates model *quality*. | Useful when deploying a new fine-tuned version to determine if it produces better responses than the current version. |
| **Data Drift** | When the distribution of incoming data changes over time, causing model performance to degrade. For example, if the SLM was fine-tuned on 2024 data but now receives questions about 2026 events. | Monitoring for data drift is a Phase 2 enhancement. When detected, it triggers model retraining. |
| **Feature Flags** | Configuration toggles that enable/disable features without deploying new code. Stored in DynamoDB or a feature flag service. | Can toggle model versions, enable/disable streaming, or activate A/B testing without a full redeployment. |

---

## Serving & Inference Terms

| Term | Definition | Why It Matters Here |
|------|------------|---------------------|
| **SLM** | Small Language Model. A language model with fewer parameters (typically 1B-7B) compared to large models (70B-400B+). Optimized for specific tasks with lower cost and latency. | The entire project is built around serving an SLM. Its smaller size means it fits on a single GPU (A10G, 24GB VRAM), dramatically reducing cost. |
| **vLLM** | An open-source inference engine for LLMs/SLMs. Uses PagedAttention for efficient GPU memory management and continuous batching for high throughput. | Our chosen inference framework. Provides 2-4x higher throughput than naive implementations and exposes an OpenAI-compatible API. |
| **TGI** | Text Generation Inference. Hugging Face's open-source inference server for LLMs. Supports FlashAttention, continuous batching, and streaming. | A viable alternative to vLLM. We chose vLLM for its superior throughput on SLMs, but TGI is a good fallback if compatibility issues arise. |
| **Triton** | NVIDIA Triton Inference Server. A production inference server supporting multiple frameworks (PyTorch, TensorFlow, TensorRT). Highly configurable but more complex to operate. | Best for multi-model, multi-framework serving. Overkill for our single-model SLM use case, but worth considering if the client adds more models later. |
| **PagedAttention** | A memory management technique used by vLLM. Manages GPU memory like an operating system manages RAM — in "pages" that can be allocated and freed dynamically. Reduces memory waste by up to 90%. | Enables serving more concurrent requests on a single GPU. Without it, each request reserves worst-case memory, severely limiting concurrency. |
| **Continuous Batching** | A serving technique where the inference engine dynamically groups incoming requests into batches as they arrive, without waiting for a fixed batch to fill up. Requests can join and leave the batch independently. | Keeps the GPU busy at all times, maximizing throughput (tokens/second). Traditional static batching leaves the GPU idle between batches. |
| **Tokenization** | Converting raw text into numerical tokens (integers) that the model can process. Each model has its own tokenizer vocabulary (e.g., "hello" might be token 15339). | Happens at the start of every inference request. Token count directly affects latency and cost (more tokens = more GPU compute). |
| **Quantization** | Reducing the numerical precision of model weights (e.g., from 16-bit floats to 4-bit integers). Reduces memory usage and increases inference speed, with minimal quality loss. | AWQ and GPTQ quantization can reduce the SLM's VRAM usage by ~50%, enabling larger context windows or more concurrent requests on the same GPU. |
| **AWQ** | Activation-Aware Weight Quantization. A quantization method that identifies which weights are most important and preserves their precision while aggressively quantizing others. | One of the best quantization methods for SLMs — minimal quality loss with significant memory savings. vLLM supports AWQ natively. |
| **GPTQ** | A post-training quantization method that compresses model weights using approximations based on the Hessian matrix (second-order information). | An alternative to AWQ. Slightly slower to apply but widely supported. Good as a fallback quantization option. |
| **Context Window** | The maximum number of tokens a model can process in a single request (input + output combined). For example, a 4096-token context window means the prompt + response can be at most 4096 tokens. | Set to 4096 in our config. Limits how long prompts and responses can be. Longer context windows use more GPU memory. |
| **Streaming (SSE)** | Server-Sent Events. A technique where the server sends generated tokens to the client one at a time as they're produced, rather than waiting for the full response. Creates a "typing" effect. | Improves perceived latency — the user sees the first word within ~50ms, even if the full response takes 2 seconds. Enabled via the `/v1/predict/stream` endpoint. |
| **Cold Start** | The time it takes for a new inference instance to become ready to serve requests. Includes: container pull, model weight download from S3, model loading into GPU memory. | For SLMs on `ml.g5.xlarge`, cold start is typically 30-60 seconds. Mitigated by keeping minimum 1 instance always running and using async pre-warming. |
| **Safetensors** | A safe, fast file format for storing model weights. Developed by Hugging Face as a more secure alternative to Python's pickle format (which can contain arbitrary code). | Our model artifacts use `.safetensors` format. It's faster to load than pickle-based `.bin` files and eliminates the risk of arbitrary code execution during model loading. |

---

## Latency & Performance Terms

| Term | Definition | Why It Matters Here |
|------|------------|---------------------|
| **Latency** | The time between sending a request and receiving a response. Measured in milliseconds (ms). Lower is better. | Our SLA targets: P50 ≤ 200ms, P99 ≤ 500ms for 128-token responses. Directly affects user experience. |
| **Throughput** | The number of requests (or tokens) the system can process per unit of time. Measured in requests/second or tokens/second. Higher is better. | Target: ≥ 100 req/s sustained. Throughput determines how many users the system can serve simultaneously. |
| **P50 / P99** | Percentile latency metrics. P50 = the latency at which 50% of requests are faster (median). P99 = the latency at which 99% of requests are faster (only 1% are slower). | P50 shows typical performance; P99 shows worst-case performance. We alarm on P99 because even rare slow requests impact user experience. |
| **TTFT** | Time to First Token. The time between sending a request and receiving the first generated token. Critical for streaming responses. | For streaming mode, TTFT determines how quickly the user sees the first word. Target: ≤ 50ms. |
| **Tokens/Second** | The rate at which the model generates output tokens. A key efficiency metric for LLM/SLM serving. | Our primary throughput metric. vLLM's continuous batching maximizes this across concurrent requests. |
| **SLA** | Service Level Agreement. A formal commitment on performance, availability, and reliability. Defines what "good enough" means. | Our SLA: 99.9% uptime, P99 < 500ms. Breaching the SLA triggers alarms and may require incident response. |
| **RTO** | Recovery Time Objective. The maximum acceptable time to restore service after a failure. | Target: ≤ 15 minutes. Auto-scaling + blue/green deployments enable fast recovery. |
| **RPO** | Recovery Point Objective. The maximum acceptable data loss in time. | 0 for inference (stateless). Logs in S3 may have up to 1-minute lag due to batching. |

---

## CI/CD & DevOps Terms

| Term | Definition | Why It Matters Here |
|------|------------|---------------------|
| **CI** | Continuous Integration. The practice of automatically building and testing code every time a change is pushed. Catches bugs early. | Every push triggers linting, unit tests, and container builds. Broken code never reaches staging or production. |
| **CD** | Continuous Deployment/Delivery. Automatically deploying tested code to staging and production environments. | After CI passes, the pipeline automatically deploys to staging, runs integration tests, and (with approval) deploys to production. |
| **IaC** | Infrastructure as Code. Defining infrastructure (servers, networks, databases) in code files instead of manually clicking through the AWS console. | All infrastructure is defined in Terraform. This means deployments are repeatable, reviewable, and version-controlled — just like application code. |
| **Terraform** | An open-source IaC tool by HashiCorp. Uses HCL (HashiCorp Configuration Language) to define infrastructure. Provider-agnostic (works with AWS, Azure, GCP). | Our chosen IaC tool. Modules in `infrastructure/modules/` define each component. `terraform plan` shows what will change before you apply it. |
| **Docker** | A platform for building, shipping, and running applications in containers. A container packages code + dependencies into a single portable unit. | The vLLM inference server runs inside a Docker container. This ensures the production environment matches development exactly — no "works on my machine" problems. |
| **ECR Image Scanning** | Automated analysis of Docker images stored in ECR to identify known software vulnerabilities (CVEs). | Catches security vulnerabilities in dependencies before they reach production. Critical for enterprise security compliance. |
| **CVE** | Common Vulnerabilities and Exposures. A standardized identifier for publicly known security vulnerabilities (e.g., CVE-2024-12345). | Our pipeline blocks deployment if the container image contains Critical or High CVEs. This prevents deploying known-vulnerable software. |
| **Buildspec** | The configuration file (`buildspec.yml`) that tells AWS CodeBuild what commands to run during each build phase. | Defines the exact steps: install dependencies → lint → test → build Docker image → push to ECR. See Section 4.1 of the LLD. |
| **Approval Gate** | A manual checkpoint in the pipeline that pauses deployment and waits for a human to approve before proceeding. | Required before production deployment. Gives the team a chance to review staging test results and decide whether to proceed. |
| **Distroless** | Minimal Docker base images that contain only the application runtime (no shell, package managers, or other OS tools). Reduces the attack surface. | Using distroless base images means there are fewer components that can have vulnerabilities, improving security posture. |

---

## Networking Terms

| Term | Definition | Why It Matters Here |
|------|------------|---------------------|
| **CIDR** | Classless Inter-Domain Routing. A notation for specifying IP address ranges (e.g., `10.0.0.0/16` = 65,536 IP addresses). | Defines the IP ranges for our VPC and subnets. Understanding CIDR is essential for designing the network layout. |
| **Subnet** | A subdivision of a VPC's IP range. Resources in a subnet share network configuration (routing, access rules). **Public subnets** have internet access; **private subnets** do not. | SageMaker endpoints run in private subnets (no internet access). The ALB runs in public subnets (accessible from the internet). This separation is core to the security architecture. |
| **Security Group** | A virtual firewall that controls inbound and outbound traffic for AWS resources. Rules specify allowed protocols, ports, and source/destination IPs. | Each component has its own Security Group with minimal allowed traffic. SageMaker instances only accept traffic from the ALB, not from the internet. |
| **IGW** | Internet Gateway. Connects a VPC to the internet. Resources in public subnets with public IPs can be reached from the internet. | Allows the ALB (and NAT Gateway) to be reachable from the internet. Private subnets do NOT have a route to the IGW. |
| **ENI** | Elastic Network Interface. A virtual network card attached to an AWS resource. VPC Endpoints create ENIs in your VPC to provide private connectivity to AWS services. | VPC Endpoint ENIs allow SageMaker instances to reach S3 and ECR without traversing the internet. |
| **TLS** | Transport Layer Security. The encryption protocol that protects data in transit (the "S" in HTTPS). TLS 1.3 is the latest and most secure version. | All external communication uses TLS 1.3. Ensures that prompts and model responses cannot be intercepted in transit. |

---

## Observability Terms

| Term | Definition | Why It Matters Here |
|------|------------|---------------------|
| **Metric** | A numerical measurement of system behavior over time (e.g., latency = 187ms, GPU utilization = 72%). | Metrics are the foundation of monitoring. Without them, you're flying blind. |
| **Alarm** | An automated alert that triggers when a metric breaches a defined threshold for a specified duration. | Alarms notify the team when things go wrong (high latency, high error rate) before users notice. |
| **Dashboard** | A visual display of key metrics, typically updated in real-time. Provides at-a-glance system health. | The CloudWatch dashboard shows inference latency, throughput, GPU utilization, error rates, and cost — all in one view. |
| **Distributed Tracing** | Tracking a single request as it flows through multiple services. Each service adds its timing information to the trace. | X-Ray traces show the latency breakdown: API Gateway (5ms) → ALB (2ms) → SageMaker (180ms). Pinpoints exactly where slowdowns occur. |
| **Log Aggregation** | Collecting logs from all components into a centralized location for searching, filtering, and analysis. | CloudWatch Logs aggregates logs from SageMaker, API Gateway, ALB, and CodeBuild. One place to search when debugging. |
| **Composite Alarm** | A CloudWatch alarm that combines multiple child alarms using AND/OR logic. Triggers only when a specific combination of conditions is true. | Example: Alert only when BOTH latency is high AND error rate is high (not just one). Reduces false positives. |

---

> [!TIP]
> **How to use this glossary:** When reading the other artifacts and you encounter an unfamiliar term, search for it here. Each entry explains not just *what* the term means, but *why* it's relevant to **your specific project**. As you grow more comfortable with these concepts, this glossary becomes less necessary — that's the goal.
