# Pilot SLM Inference Plan

> **Status:** Internal Planning Document  
> **Objective:** Prove the core end-to-end functionality of serving an SLM on AWS for a rapid client demonstration, before investing time in the full enterprise-scale architecture.

---

## 🎯 Purpose of the Pilot

The full enterprise architecture (with CI/CD pipelines, multi-AZ networking, WAF, blue/green deployments, etc.) is highly robust but takes time to provision. 

**This Pilot is designed to:**
1. **Prove the Core Tech:** Demonstrate that `vLLM` can successfully serve our custom model weights on an AWS GPU instance.
2. **Validate Latency:** Get baseline metrics for token-generation latency.
3. **Showcase the Integration:** Allow the client to hit a real AWS API Gateway endpoint from Postman or their own application.
4. **Secure Buy-In:** Serve as a tangible milestone to secure approval to build out the rest of the MLSecOps landscape.

---

## ⚖️ Scope & Landscape

To achieve a rapid deployment (within a 3-week pilot sprint rather than a multi-month rollout), we will strip back the enterprise layers and focus strictly on the inference path.

### What is IN Scope (The Pilot):
- Manual creation of a single Amazon SageMaker Real-Time Endpoint (GPU instance selected based on the model's VRAM and parameter requirements).
- Manual upload of a basic `vLLM` Docker image to ECR.
- Manual upload of model weights (`.safetensors`) to an S3 bucket.
- A simple API Gateway (HTTP API) directly invoking the SageMaker endpoint.
- Basic API Key authentication.

### What is OUT of Scope (Saved for Phase 2):
- ❌ **CI/CD Automation:** No CodePipeline or CodeBuild. We will run Docker builds locally.
- ❌ **Advanced Security:** No WAF, no strict private subnets, no IAM granular least-privilege (we will use managed policies for speed).
- ❌ **Load Balancing:** No Application Load Balancer (ALB) or Auto Scaling Groups.
- ❌ **Observability:** No custom CloudWatch dashboards or alarms.

---

## 🏗️ Pilot Architecture

Notice how simplified this architecture is compared to the final enterprise version. It represents the absolute minimum viable path from a user request to a GPU calculation.

![Pilot SLM Inference Architecture](images/pilot_architecture.png)

---

## 🏃 Execution Plan (3-Week Sprint)

### Phase 1: Local Container Validation & Setup (Week 1)
- **Goal:** Prove the container works before dealing with AWS.
- **Action:** Write the `vLLM` Dockerfile. Run the container locally using a dummy model (e.g., a tiny HuggingFace model) to ensure the server starts and responds to HTTP requests. Procure necessary AWS credentials.

### Phase 2: Manual AWS Infrastructure & Deployment (Week 2)
- **Goal:** Get the model running in the cloud.
- **Action:** 
  1. Create an S3 bucket and upload the real custom SLM weights.
  2. Create an ECR repository and push our local Docker image.
  3. Go to the AWS Console → SageMaker → Create Model (using the ECR image and S3 data).
  4. Create an Endpoint Configuration.
  5. Deploy the Endpoint and run internal connectivity tests.

### Phase 3: The Front Door & Client Demonstration (Week 3)
- **Goal:** Expose the endpoint securely and prove success to stakeholders.
- **Action:**
  1. Create a simple integration to connect API Gateway to the SageMaker Endpoint.
  2. Generate an API Key for the client.
  3. Provide the client with the API Endpoint URL and the API Key. 
  4. Run a live demonstration showing the input prompt and the generated response, highlighting the latency (Time to First Token).

---

## ✅ Success Criteria for the Pilot
- [ ] Endpoint successfully responds with a `200 OK` and valid JSON payload.
- [ ] Time to First Token (TTFT) is under 500ms.
- [ ] Client can successfully authenticate via API Key.
- [ ] Model accurately reflects the fine-tuned behavior (not just the base model).
