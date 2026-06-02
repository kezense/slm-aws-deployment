# 2. Product Requirements Document (PRD)

> **Document Version:** 1.0  
> **Status:** Draft  
> **Last Updated:** 2026-05-31  
> **Owner:** Ashwin Beryl Kalaichandran

---

## 2.1 Executive Summary

### What Are We Building?

A **production-grade, secure inference platform** on AWS for serving a client's custom fine-tuned Small Language Model (SLM). The platform will provide:

- A **RESTful API endpoint** that accepts natural language prompts and returns model-generated responses with sub-second latency.
- A **fully automated CI/CD pipeline** that handles the entire model lifecycle — from code commit to production deployment — with zero manual intervention.
- **Enterprise-grade security** including encryption at rest and in transit, network isolation, vulnerability scanning, and comprehensive audit logging.

### Why Are We Building It?

| Driver | Detail |
|--------|--------|
| **Business Need** | The client has invested in fine-tuning a domain-specific SLM and needs it deployed as a reliable, scalable service that internal applications and end-users can consume. |
| **Security Mandate** | The model processes sensitive domain data. Deployment must meet enterprise security standards — data must never traverse the public internet unencrypted, and access must be tightly controlled. |
| **Operational Efficiency** | Manual deployment of ML models is error-prone and slow. Automating the pipeline reduces deployment time from days to minutes and eliminates human error. |
| **Cost Optimization** | SLMs are significantly cheaper to serve than large LLMs. The architecture must leverage this advantage with right-sized GPU instances and auto-scaling to minimize cloud spend. |

---

## 2.2 Scope

### ✅ In Scope

| Area | Details |
|------|---------|
| **Model Serving** | Deploy the pre-fine-tuned SLM as a real-time inference endpoint on AWS |
| **CI/CD Pipeline** | Automated pipeline: build → test → scan → register → deploy |
| **Infrastructure as Code** | All infrastructure defined in Terraform (or AWS CDK) — nothing click-ops |
| **Security** | VPC isolation, IAM least-privilege, KMS encryption, WAF, Secrets Manager |
| **Observability** | CloudWatch metrics, alarms, dashboards for latency, throughput, errors, GPU utilization |
| **Auto-Scaling** | Scale inference endpoints based on request volume and GPU utilization |
| **Blue/Green Deployment** | Zero-downtime deployments with automatic rollback on failure |
| **Model Registry** | Version-controlled model artifacts with approval gates |
| **API Management** | API Gateway with rate limiting, authentication, and usage plans |

### ❌ Out of Scope

| Area | Rationale |
|------|-----------|
| **Model Training / Fine-Tuning** | The model is already fine-tuned. Training infrastructure is a separate workstream. |
| **Data Pipeline / ETL** | Data ingestion, preprocessing, and feature engineering are handled by a separate team. |
| **Frontend / UI** | This project delivers an API — front-end integration is handled by the consuming application teams. |
| **Multi-Region Deployment** | V1 is single-region. Multi-region DR will be addressed in V2 based on usage patterns. |
| **Model Retraining Automation** | Automated retraining triggers (data drift detection, etc.) are a Phase 2 enhancement. |
| **Cost Allocation / Chargebacks** | Tagging strategy will be defined but cross-team cost allocation is out of scope. |

---

## 2.3 Functional Requirements

### FR-1: Model Serving Endpoint

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-1.1 | The system SHALL expose a REST API endpoint (`POST /v1/predict`) that accepts a JSON payload containing a text prompt | **P0** |
| FR-1.2 | The endpoint SHALL return a JSON response containing the generated text, token count, and latency metadata | **P0** |
| FR-1.3 | The endpoint SHALL support streaming responses via Server-Sent Events (SSE) for long-form generation | **P1** |
| FR-1.4 | The endpoint SHALL accept optional parameters: `max_tokens`, `temperature`, `top_p`, `stop_sequences` | **P1** |

### FR-2: CI/CD Pipeline

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-2.1 | A code push to the `main` branch SHALL trigger the CI/CD pipeline automatically | **P0** |
| FR-2.2 | The pipeline SHALL run linting, unit tests, and integration tests before deployment | **P0** |
| FR-2.3 | The pipeline SHALL build and push a Docker container image to ECR | **P0** |
| FR-2.4 | The pipeline SHALL register the model version in SageMaker Model Registry | **P0** |
| FR-2.5 | The pipeline SHALL deploy to a staging environment before production | **P0** |
| FR-2.6 | The pipeline SHALL support manual approval gates before production deployment | **P1** |

### FR-3: Model Lifecycle Management

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-3.1 | The system SHALL maintain a versioned registry of all deployed model artifacts | **P0** |
| FR-3.2 | The system SHALL support rollback to any previously deployed model version within 5 minutes | **P0** |
| FR-3.3 | The system SHALL track model lineage (which training data, hyperparameters, and code produced each version) | **P1** |

### FR-4: Authentication & Authorization

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-4.1 | All API requests SHALL be authenticated via API keys or JWT tokens | **P0** |
| FR-4.2 | The system SHALL support multiple API key tiers with different rate limits | **P1** |
| FR-4.3 | The system SHALL log all authentication attempts (success and failure) | **P0** |

---

## 2.4 Non-Functional Requirements

### NFR-1: Performance

| Metric | Target | Measurement Method |
|--------|--------|--------------------|
| **P50 Latency** (end-to-end) | ≤ 200ms for 128-token response | CloudWatch percentile metrics |
| **P99 Latency** (end-to-end) | ≤ 500ms for 128-token response | CloudWatch percentile metrics |
| **Throughput** | ≥ 100 requests/second sustained | Load testing with Locust/k6 |
| **Token Generation Rate** | ≥ 50 tokens/second per request | Custom CloudWatch metric |
| **Cold Start Time** | ≤ 60 seconds (new instance) | SageMaker endpoint logs |

> [!NOTE]
> **Latency vs. Throughput:** Latency is *how fast* a single request completes. Throughput is *how many* requests the system handles per second. You need to optimize for both — a system can have low latency but also low throughput if it can only handle one request at a time. The serving framework (vLLM) uses **continuous batching** to maximize throughput without sacrificing latency.

### NFR-2: Availability & Reliability

| Metric | Target |
|--------|--------|
| **Uptime SLA** | 99.9% (≤ 8.76 hours downtime/year) |
| **Recovery Time Objective (RTO)** | ≤ 15 minutes |
| **Recovery Point Objective (RPO)** | 0 (stateless serving — no data loss) |
| **Deployment Downtime** | Zero (blue/green deployments) |

### NFR-3: Security

| Requirement | Implementation |
|-------------|---------------|
| **Encryption at Rest** | All S3 buckets, ECR images, and EBS volumes encrypted with AWS KMS (customer-managed keys) |
| **Encryption in Transit** | TLS 1.3 enforced on all endpoints; internal traffic uses VPC-internal encryption |
| **Network Isolation** | Model endpoints run in private subnets with no direct internet access |
| **Least Privilege IAM** | Each service component has its own IAM role with minimal required permissions |
| **Vulnerability Scanning** | Automated CVE scanning on every container image push (ECR + Amazon Inspector) |
| **Audit Logging** | All API calls logged via CloudTrail; all data access logged via S3 access logs |
| **Secrets Management** | No hardcoded credentials — all secrets stored in AWS Secrets Manager |

### NFR-4: Cost Optimization

| Strategy | Expected Impact |
|----------|----------------|
| **Right-sized GPU instances** | Use `ml.g5.xlarge` (single A10G GPU) for SLMs instead of expensive multi-GPU instances | 
| **Auto-scaling to zero** | Scale down inference instances during off-peak hours (if async mode is acceptable) |
| **Spot instances for staging** | Use Spot instances for non-production environments (up to 90% savings) |
| **S3 Intelligent Tiering** | Automatically move old model artifacts to cheaper storage classes |
| **Reserved capacity** | Purchase Savings Plans for predictable production workloads |

---

## 2.5 Success Metrics

| Category | Metric | Target | Timeline |
|----------|--------|--------|----------|
| **Deployment Velocity** | Time from code commit to production | ≤ 30 minutes | End of Production (Week 9) |
| **Reliability** | Successful deployment rate | ≥ 95% | End of Production (Week 9) |
| **Performance** | P99 latency under load | ≤ 500ms | End of Pilot (Week 3) |
| **Security** | Zero critical/high CVEs in production images | 0 CVEs | Ongoing |
| **Cost** | Monthly inference cost per 1M requests | ≤ $500 | End of Production (Week 9) |
| **Operational** | Mean Time to Recovery (MTTR) | ≤ 15 minutes | End of Production (Week 9) |
| **Automation** | Manual intervention required per deployment | 0 steps | End of Production (Week 9) |

---

## 2.6 Stakeholder Matrix

| Stakeholder | Role | Interest |
|-------------|------|----------|
| **Client Engineering Team** | Consumer of the API | Low-latency, reliable endpoint with clear documentation |
| **Client Security Team** | Compliance & audit | Encryption, access controls, audit trails |
| **Client Data Science Team** | Model owners | Model versioning, easy rollback, experiment tracking |
| **Platform Team (You)** | Builder & operator | Automated deployment, observability, maintainability |

---

## 2.7 Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| GPU instance unavailability in chosen region | Medium | High | Configure fallback instance types; request capacity reservations |
| Model latency exceeds SLA under load | Medium | High | Load test early; implement request queuing and auto-scaling policies |
| Security vulnerability in base container image | Low | Critical | Automated scanning in CI/CD; use minimal base images (distroless) |
| Cost overrun from auto-scaling | Medium | Medium | Set hard scaling limits; implement CloudWatch billing alarms |
| Model quality regression after deployment | Low | High | Shadow deployment + A/B testing before full traffic cutover |

> [!IMPORTANT]
> This PRD defines the **"what"** and **"why"**. The following artifacts (HLD and LLD) define the **"how"**. Every technical decision in the HLD and LLD should trace back to a requirement in this document.
