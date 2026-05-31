# SLM AWS Deployment Infrastructure

Welcome to the SLM AWS Deployment repository. This project contains the complete end-to-end architecture, design specifications, and infrastructure configurations necessary for deploying a custom fine-tuned Small Language Model (SLM) securely into an AWS environment.

## 🎯 Repository Intent

The primary goal of this repository is to provide a production-grade, highly secure, and automated deployment strategy for serving an SLM. Key focuses include:
- **Automation:** A fully automated CI/CD pipeline from code commit to model serving.
- **Security:** Enterprise-grade network isolation (VPC, Private Subnets), strict IAM least-privilege policies, AWS WAF, and encryption.
- **Performance:** High-performance inference endpoints utilizing `vLLM` on Amazon SageMaker for continuous batching and maximum token throughput.
- **Reliability:** Robust observability, auto-scaling, and zero-downtime blue/green deployments.

## 📚 Documentation Navigation

If you are looking for the rapid deployment strategy to demonstrate functionality before building the full enterprise architecture, please see the **[Pilot SLM Inference Plan](./docs/pilot_project_plan.md)**.

The full enterprise architecture and design strategies are meticulously detailed in the `docs/` directory. For a complete understanding of the system, we recommend reading through them in the following logical order:

| # | Document | Description |
|---|---|---|
| **1** | [Architecture Diagrams](./docs/1_architecture_diagrams.md) | Visual blueprints of the system (CI/CD flow, layered AWS infrastructure, request path). |
| **2** | [Product Requirements (PRD)](./docs/2_product_requirements_document.md) | The "What" and "Why": scope, functional/non-functional requirements, SLA targets, and success metrics. |
| **3** | [High-Level Design (HLD)](./docs/3_high_level_design.md) | The conceptual "How": AWS service justifications, model lifecycle management, and environment strategies. |
| **4** | [Low-Level Design (LLD)](./docs/4_low_level_design.md) | The technical "How": detailed configurations, pipeline scripts, subnet allocation, WAF rules, and IAM policies. |
| **5** | [MLOps & AWS Glossary](./docs/5_mlops_aws_glossary.md) | A curated dictionary defining all AWS services and MLOps concepts used across these documents. |

