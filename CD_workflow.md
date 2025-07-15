As an expert in DevOps, CI/CD, and Spinnaker, I can provide you with a detailed, production-level runbook for a Continuous Deployment (CD) workflow using Spinnaker. This workflow begins *after* your CI pipeline has successfully completed and produced a deployable artifact, typically a Docker image.

The core objective of a production-level CD pipeline with Spinnaker is **safe, automated, and observable delivery** to production with minimal human intervention and maximum confidence. This means:

*   **Automated Verification:** Relying heavily on automated tests and health checks at each stage.
*   **Progressive Delivery:** Gradually exposing new code to real traffic (e.g., Canary deployments).
*   **Built-in Rollback:** The ability to quickly and reliably revert to a previous stable state.
*   **Auditability:** A clear record of who, what, when, and where a deployment occurred.

---

### **Production-Level CD Workflow: Spinnaker Pipeline Runbook**

**Application:** `product-catalog-service` (deployed as a Docker container)
**Artifact Source:** Docker Image pushed to a Container Registry (e.g., GCR, ECR, ACR, Artifactory)
**Deployment Target:** Kubernetes Cluster (GKE, EKS, AKS, or on-prem)
**Monitoring & Metrics:** Prometheus/Datadog/New Relic (for Kayenta analysis)
**Notification Channel:** Slack

---

#### **Runbook: Continuous Deployment Pipeline for `product-catalog-service` (Spinnaker)**

**1. Purpose:**
To automate the safe and progressive deployment of new versions of the `product-catalog-service` microservice across development, staging, and production environments, ensuring high availability and quick detection/mitigation of issues.

**2. Pre-requisites:**
*   A stable Spinnaker instance configured with:
    *   Access to your Kubernetes clusters (dev, staging, prod).
    *   Integration with your Container Registry for artifact polling.
    *   Kayenta configured with your metrics provider (Prometheus, Datadog, etc.).
    *   Configured notification channels (Slack, email).
    *   Necessary IAM roles/service accounts for Spinnaker to deploy to Kubernetes and interact with monitoring tools.
*   CI Pipeline successfully producing and pushing a Docker image tagged with a unique, immutable identifier (e.g., `product-catalog-service:1.2.3-abcd123`).
*   Kubernetes manifests (Deployments, Services, Ingress/Gateway, HPA) for `product-catalog-service` stored in Git.
*   Automated tests (smoke, integration, E2E) available to run against deployed environments.

**3. Spinnaker Pipeline Definition:**
The pipeline will be defined declaratively (e.g., using Spinnaker's Managed Delivery features or directly via the UI/JSON export) and typically named `product-catalog-service-cd`.

**4. Trigger:**
*   **Type:** `Artifact` (Docker Registry)
*   **Details:** Configured to monitor the specific container registry path for new images matching `product-catalog-service:*`.
*   **Start Pipeline:** Automatically triggers upon discovery of a new image.

**5. Pipeline Stages (Detailed Steps):**

---

#### **Stage 1: Deploy to Development Environment**

*   **Type:** `Deploy (Manifest)`
*   **Purpose:** Initial deployment to a low-risk, developer-facing environment for rapid feedback and integration testing.
*   **Configuration:**
    *   **Account:** `kubernetes-dev`
    *   **Namespace:** `dev-product-catalog`
    *   **Manifest Source:** Git (e.g., `k8s/dev/product-catalog-service.yaml`) or Artifact (if manifests are templated/generated in CI).
    *   **Manifest Override:** Inject the new Docker image tag (`product-catalog-service:${trigger.artifacts[0].tag}`) into the Deployment manifest.
    *   **Strategy:** `Rolling Update` (fast feedback is prioritized here).
    *   **Traffic Management:** Standard Kubernetes Service/Ingress.
*   **Post-Deployment Actions:**
    *   **Type:** `Wait` (e.g., 60 seconds) to allow pods to start.
    *   **Type:** `Check Health` (Kubernetes health checks)
    *   **Type:** `Run Job (Manifest)`: Execute automated smoke tests / basic integration tests against the `dev-product-catalog` endpoint. This could be a Kubernetes Job running a containerized test suite.
        *   **On Failure:** If tests fail, send an immediate Slack notification and stop the pipeline.

---

#### **Stage 2: Manual Judgment - Dev Verification**

*   **Type:** `Manual Judgment`
*   **Purpose:** Allow developers to visually inspect the service in the Dev environment and perform quick sanity checks. This is often skipped in highly mature setups but can be useful for new services or teams.
*   **Configuration:**
    *   **Message:** "Product Catalog Service ${trigger.artifacts.tag} deployed to Dev. Please verify: [Link to Dev UI], [Link to Dev Logs]"
    *   **Input Options:** "Approve", "Reject"
    *   **Restrict to Users/Groups:** `devops-team`, `product-catalog-devs`
*   **On Rejection:** Terminate pipeline.

---

#### **Stage 3: Deploy to Staging Environment (Blue/Green)**

*   **Type:** `Deploy (Manifest)`
*   **Purpose:** Deploy to a production-like staging environment for comprehensive integration and end-to-end testing, mirroring production behavior.
*   **Configuration:**
    *   **Account:** `kubernetes-staging`
    *   **Namespace:** `staging-product-catalog`
    *   **Manifest Source:** Git (e.g., `k8s/staging/product-catalog-service.yaml`)
    *   **Manifest Override:** Inject new Docker image tag.
    *   **Strategy:** `Blue/Green` (for safer deployment in a critical environment).
        *   Spinnaker deploys the *new* version (Green) alongside the *old* version (Blue).
        *   **Traffic Management:** Spinnaker manages the Service selector to point to the active (Blue initially) or new (Green) ReplicaSet.
*   **Post-Deployment Actions (Green Verification):**
    *   **Type:** `Wait` (e.g., 90 seconds).
    *   **Type:** `Check Health` (Kubernetes health checks on Green pods).
    *   **Type:** `Run Job (Manifest)`: Execute comprehensive E2E tests against the *newly deployed Green environment* (e.g., via a dedicated staging ingress for Green, or direct service access if possible). These tests validate all critical user flows.
        *   **On Failure:** If E2E tests fail, send Slack notification, scale down Green, and terminate pipeline.

---

#### **Stage 4: Manual Judgment - Staging Approval**

*   **Type:** `Manual Judgment`
*   **Purpose:** Critical gate before production. Requires explicit sign-off from QA, Product, or Lead Engineers after E2E tests and manual exploratory testing on Staging.
*   **Configuration:**
    *   **Message:** "Product Catalog Service ${trigger.artifacts.tag} passed Staging E2E tests. Ready for Production deployment? Verify Staging: [Link to Staging UI], [Link to Staging Metrics], [Link to Test Reports]"
    *   **Input Options:** "Approve Production Deployment", "Reject"
    *   **Restrict to Users/Groups:** `qa-leads`, `product-owners`, `release-managers`
*   **On Rejection:** Terminate pipeline. If approved, Spinnaker proceeds to shift traffic on Staging.
*   **Internal Action (Spinnaker):** Upon approval, Spinnaker performs the atomic traffic cutover on the Staging environment (switches `Service` selector from Blue to Green).

---

#### **Stage 5: Canary Analysis (Kayenta) - Production Readiness**

*   **Type:** `Canary Analysis`
*   **Purpose:** Safely introduce the new version to a small subset of production traffic and automatically compare its performance and behavior against the stable baseline.
*   **Configuration:**
    *   **Account:** `kubernetes-prod`
    *   **Namespace:** `prod-product-catalog`
    *   **Baseline/Canary Selection:** Spinnaker automatically identifies the current production ReplicaSet as the "baseline" and deploys a small "canary" ReplicaSet (e.g., 5-10% of desired production replicas) with the *new* image.
    *   **Traffic Routing:** Spinnaker integrates with your Load Balancer (e.g., Kubernetes Ingress, Istio, Nginx, AWS ALB) to send a percentage of *live production traffic* to the canary.
    *   **Metric Sources:** Define metrics for comparison (e.g., `request.latency`, `http.5xx_errors`, `cpu.usage`, `memory.usage`, `database.errors`, `business.conversion_rate`).
        *   **Metrics Provider:** `Prometheus` (or `Datadog`, `Stackdriver`, etc.)
        *   **Query Templates:** Define queries for baseline and canary.
    *   **Analysis Configuration:**
        *   **Duration:** 15-30 minutes (depends on traffic volume and observation period needed).
        *   **Interval:** 1 minute.
        *   **Score Thresholds:** `Pass` (e.g., 90%), `Marginal` (e.g., 75%), `Fail` (e.g., < 75%).
        *   **Automated Action:** `Automate Canary Result` enabled.
            *   **Success Action:** `Continue Pipeline`
            *   **Marginal Action:** `Manual Judgment` (or `Rollback Canary` based on risk appetite)
            *   **Fail Action:** `Rollback Canary`
*   **On Canary Failure/Marginal:**
    *   Spinnaker automatically scales down and deletes the canary ReplicaSet.
    *   Sends an immediate Slack alert with Kayenta report link.
    *   Pipeline terminates (or awaits manual judgment for marginal).

---

#### **Stage 6: Deploy to Production (Blue/Green)**

*   **Type:** `Deploy (Manifest)`
*   **Purpose:** Full rollout of the new version to production, leveraging the Blue/Green strategy for zero-downtime and instant rollback capability.
*   **Configuration:**
    *   **Account:** `kubernetes-prod`
    *   **Namespace:** `prod-product-catalog`
    *   **Manifest Source:** Git (e.g., `k8s/prod/product-catalog-service.yaml`)
    *   **Manifest Override:** Inject the new Docker image tag.
    *   **Strategy:** `Blue/Green`
        *   Spinnaker deploys the full desired count of *new* version pods (Green) alongside the *current* production (Blue).
        *   **Traffic Management:** Once Green pods are healthy, Spinnaker atomically updates the Kubernetes Service selector to point to the Green ReplicaSet, effectively shifting all production traffic to the new version.
*   **Post-Deployment Verification (Green is Live):**
    *   **Type:** `Wait` (e.g., 60 seconds) for traffic to stabilize.
    *   **Type:** `Check Health` (final health check on live Green pods).
    *   **Type:** `Run Job (Manifest)`: Run a small set of post-deployment smoke tests or critical API checks against the live production endpoint to ensure everything is functional after traffic shift.
        *   **On Failure:** Immediately trigger the **Rollback to Previous Production** stage.

---

#### **Stage 7: Post-Deployment Actions & Cleanup**

*   **Type:** `Wait` (e.g., 5-10 minutes - observation window for any immediate issues).
*   **Type:** `Disable Old Server Group (Blue)`
    *   **Purpose:** After a successful cutover and observation period, the old "Blue" production environment is disabled but not immediately deleted. This allows for rapid rollback if an issue surfaces hours later.
*   **Type:** `Clean Up Old Server Groups`
    *   **Purpose:** After a longer grace period (e.g., 24-48 hours, or a manual decision), the old disabled server group is removed to reclaim resources. This is often a separate, scheduled cleanup pipeline, or a manual trigger.
*   **Type:** `Slack Notification`
    *   **Message:** "Product Catalog Service ${trigger.artifacts.tag} **LIVE in Production**! Monitoring: [Link to Production Dashboard], [Link to Production Logs]"

---

#### **Rollback Strategy (Built into Spinnaker)**

Spinnaker inherently supports rollbacks for Blue/Green and Canary deployments. If any stage (Canary analysis, Post-Deployment verification) fails:

*   **Automatic Rollback (for Canary):** Kayenta's failure action will automatically scale down the canary and revert traffic.
*   **Manual/Automated Rollback (for Blue/Green):** If the production cutover or post-deployment checks fail, a Spinnaker `Rollback` stage (triggered by an on-failure condition) can simply switch the Kubernetes Service selector back to the previously stable "Blue" environment, providing an instant reversion.
*   **Manual Rollback (any time):** Any previous successful deployment (server group) can be manually selected in Spinnaker and deployed as a rollback.

---

### **Key Production Considerations for Spinnaker CD:**

1.  **Observability Integration:** Spinnaker's power is amplified by robust integration with your monitoring, logging, and tracing solutions. Metrics are crucial for Kayenta; logs and traces are vital for debugging.
2.  **Secrets Management:** Never hardcode secrets. Spinnaker should integrate with external secrets managers (e.g., Vault, AWS Secrets Manager, GCP Secret Manager, Azure Key Vault) to inject sensitive information into deployments.
3.  **Permissions & Least Privilege:** Configure Spinnaker's RBAC (Role-Based Access Control) carefully. Limit who can deploy to which environments and what actions they can perform.
4.  **Pipeline as Code / Managed Delivery:** For true enterprise scale, define your Spinnaker pipelines as code (e.g., JSON or YAML managed in Git) using Spinnaker's Managed Delivery (formerly Spinnaker as Code or Armory's Dinghy). This ensures version control, review, and easier management of many pipelines.
5.  **Environment Parity:** Strive for maximum parity between staging and production environments to minimize "it works on my machine/staging" issues.
6.  **Cost Management:** Ensure old server groups/replica sets are cleaned up automatically after a defined grace period to avoid unnecessary infrastructure costs.
7.  **Notifications & Alerts:** Configure comprehensive notifications (Slack, email, PagerDuty) for pipeline status (success, failure, manual judgment required), critical Kayenta findings, and deployment issues.
8.  **Audit Trails:** Spinnaker provides an excellent audit trail of all deployments, including who initiated them, what artifacts were deployed, and the pipeline execution details.

This detailed Spinnaker CD workflow provides a robust, automated, and secure path to production, significantly reducing deployment risk and improving release velocity for microservices.