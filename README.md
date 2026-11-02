<h1 align="center">Boutique Platform — Cloud-Native DevOps & AIOps on AWS</h1>

<p align="center">
  <img src="docs/evidence/aws-main-logo-2025-orange-arrow-icon-1.png" alt="AWS" width="120" />
</p>

<p align="center">
  <img alt="Terraform" src="https://img.shields.io/badge/Terraform-7B42BC?logo=terraform&logoColor=white">
  <img alt="Amazon EKS" src="https://img.shields.io/badge/Amazon_EKS-FF9900?logo=amazon-eks&logoColor=white">
  <img alt="Karpenter" src="https://img.shields.io/badge/Karpenter-FF9900?logo=kubernetes&logoColor=white">
  <img alt="GitHub Actions" src="https://img.shields.io/badge/GitHub_Actions-2088FF?logo=github-actions&logoColor=white">
  <img alt="Amazon ECR" src="https://img.shields.io/badge/Amazon_ECR-FF9900?logo=amazon-ecr&logoColor=white">
  <img alt="Argo CD" src="https://img.shields.io/badge/Argo_CD-EF7B4D?logo=argo&logoColor=white">
</p>
<p align="center">
  <img alt="Helm" src="https://img.shields.io/badge/Helm-0F1689?logo=helm&logoColor=white">
  <img alt="Prometheus" src="https://img.shields.io/badge/Prometheus-E6522C?logo=prometheus&logoColor=white">
  <img alt="Grafana" src="https://img.shields.io/badge/Grafana-F46800?logo=grafana&logoColor=white">
  <img alt="Jaeger" src="https://img.shields.io/badge/Jaeger-66CFE3?logo=jaeger&logoColor=black">
  <img alt="CloudWatch" src="https://img.shields.io/badge/CloudWatch-FF4F8B?logo=amazoncloudwatch&logoColor=white">
  <img alt="Bedrock" src="https://img.shields.io/badge/Bedrock-232F3E?logo=amazon-aws&logoColor=white">
</p>

End-to-end boutique platform: a multi-service store on **Amazon EKS**, provisioned with **Terraform**, shipped with **GitHub Actions** and **Argo CD**, observed with **Prometheus**, **Grafana**, **Jaeger**, and **CloudWatch**, and operated with **Kira** on Bedrock AgentCore.

> **Status:** Built and validated in production, then torn down for cost control. This repository presents the architecture and the screenshots. The source code and infrastructure stay private. Hostnames do not resolve while the stack is destroyed.

| | |
|---|---|
| **Cloud** | AWS `us-east-1` |
| **Compute** | EKS `eks-cluster-prod` (K8s 1.34) + Karpenter |
| **IaC** | Terraform (`modules` + `environments/prod`) |
| **Delivery** | GitHub Actions → ECR → Argo CD |
| **Data** | RDS PostgreSQL 16 |
| **AIOps** | AgentCore Harness/Gateway + 3 tool Lambdas |
| **State** | Torn down — source is private; this repo keeps the report and screenshots |

| Service | Hostname while the stack was up |
|---|---|
| Storefront | `app.simsoliver.com` |
| Argo CD | `argo.simsoliver.com` |
| Grafana | `graf.simsoliver.com` |
| Jaeger | `trace.simsoliver.com` |
| Kira | `kira.simsoliver.com` |

## Contents

1. [Architecture](#architecture)
   - [Platform (full stack)](#platform-full-stack)
   - [Application services](#application-services)
   - [Kira AIOps](#kira-aiops)
2. [What this demonstrates](#what-this-demonstrates)
3. [Repository layout](#repository-layout)
4. [Application](#application)
   - [Storefront](#storefront)
5. [Infrastructure (Terraform)](#infrastructure-terraform)
   - [Compute & registry evidence](#compute--registry-evidence)
6. [CI/CD](#cicd)
7. [GitOps (Argo CD)](#gitops-argo-cd)
8. [Observability](#observability)
   - [Metrics — Grafana](#metrics--grafana)
   - [Traces — Jaeger](#traces--jaeger)
   - [Logs — CloudWatch](#logs--cloudwatch)
9. [AIOps — Kira](#aiops--kira)
10. [Security & networking](#security--networking)
11. [Operate](#operate)
12. [Author](#author)

---

## Architecture

### Platform (full stack)

<p align="center">
  <img src="docs/architecture/boutique-platform-architecture-v29.png" alt="Boutique platform architecture" width="820" />
</p>

### Application services

<p align="center">
  <img src="docs/architecture/boutique-architecture-dark.png" alt="Boutique service architecture" width="820" />
</p>

**Request path**

```text
Browser → ALB /     → frontend :3000
Browser → ALB /api  → gateway  :3001 → auth | products | orders | order-service | users
orders → product-service (price) → order-service (notify)
```

### Kira AIOps

`kira-ui` (IRSA) → Harness `kira_aiops` → Gateway `aiops-kira-tools` → `aiops-fetch-logs` / `metrics` / `health` → CloudWatch · Prometheus · EKS API

<p align="center">
  <img src="docs/architecture/kira-aiops-architecture-dark.png" alt="Kira architecture" width="820" />
</p>

---

## What this demonstrates

- **IaC** — VPC (3 AZs), EKS, ECR, RDS, platform add-ons, Karpenter, AIOps
- **GitOps** — Argo CD + Kustomize under `gitops/`
- **CI/CD** — path-aware matrix builds → ECR → manifest tag updates
- **Microservices** — frontend, gateway, auth, products, orders, order-service, users
- **Secrets** — Secrets Manager → External Secrets → Kubernetes
- **Observability** — Prometheus/Grafana, Jaeger, Fluent Bit → CloudWatch
- **Reliability** — multi-AZ workers, topology spread, Horizontal Pod Autoscalers, and PodDisruptionBudgets
- **AIOps** — conversational SRE agent on Bedrock AgentCore
- **Cost control** — documented teardown / restore runbooks

---

## Repository layout

Source, GitOps, and runbooks stay in the private repository. This report keeps the diagrams and screenshots under `docs/`.

```text
.
├── projects/
│   ├── boutique-microservices/   # App (7 services)
│   ├── Infrastructure/           # Terraform modules + envs
│   └── aiops-assistant/          # Kira UI + Lambdas + deploy
├── gitops/                       # Argo CD apps, ingress, Karpenter
├── docs/
│   ├── architecture/             # Platform, service, and Kira diagrams
│   └── evidence/                 # Validation screenshots
├── SERVICE-FLOWS.txt             # Boutique request flows
├── KIRA-FLOWS.txt                # Kira tool paths
├── RESTORATION.txt / TEARDOWN.txt
└── .github/workflows/ci.yml
```

---

## Application

| Service | Port | Role | DB |
|---|---|---|---|
| `frontend` | 3000 | React SPA | — |
| `gateway` | 3001 | API proxy | — |
| `auth` | 3002 | Auth | `auth_db` |
| `product-service` | 3003 | Catalog | `products_db` |
| `order-service` | 3004 | Order processing | `orders_db` |
| `orders` | 3005 | Checkout / history | `orders_db` |
| `user-service` | 3006 | Profiles | `users_db` |

### Storefront

<table width="100%">
  <tr>
    <th colspan="2" align="center">Homepage</th>
  </tr>
  <tr>
    <td colspan="2" align="center">
      <img src="docs/evidence/boutique-storefront-homepage.png" width="520" alt="Homepage" />
    </td>
  </tr>
  <tr>
    <th align="center" width="50%">Catalog</th>
    <th align="center" width="50%">Checkout</th>
  </tr>
  <tr>
    <td align="center">
      <img src="docs/evidence/boutique-products-catalog.png" width="390" alt="Catalog" />
    </td>
    <td align="center">
      <img src="docs/evidence/boutique-checkout-page.png" width="390" alt="Checkout" />
    </td>
  </tr>
</table>

---

## Infrastructure (Terraform)

| Module | Responsibility |
|---|---|
| `vpc` | Subnets, NAT, SGs, Route 53 lookup, ACM |
| `eks` | Control plane, node group, OIDC |
| `ecr` | `prod-*` repositories |
| `boutique-db` | RDS PostgreSQL 16 + Secrets Manager |
| `eks-platform` | Argo CD, kube-prometheus, Jaeger, Fluent Bit, LBC, external-dns, ESO |
| `karpenter` | IRSA + Helm |
| `aiops-kira` | Lambdas, AgentCore, IRSA, Prometheus NLB |

Wave 1 creates the VPC, EKS cluster, ECR repositories, and RDS. Wave 2 enables the platform and AIOps flags in the prod tfvars.

### Compute & registry evidence

#### EKS workers (multi-AZ)

Managed node group across availability zones. Each panel below is the pod list for one worker.

<table width="100%">
  <tr>
    <th colspan="3" align="center" width="50%">EC2</th>
    <th colspan="3" align="center" width="50%">Compute</th>
  </tr>
  <tr>
    <td colspan="3" align="center">
      <img src="docs/evidence/eks-ec2-worker-nodes.png" width="100%" alt="EKS nodes" />
    </td>
    <td colspan="3" align="center">
      <img src="docs/evidence/eks-compute-nodes.png" width="100%" alt="EKS compute nodes" />
    </td>
  </tr>
  <tr>
    <th colspan="2" align="center" width="33%">10.2.11.80</th>
    <th colspan="2" align="center" width="33%">10.2.11.80</th>
    <th colspan="2" align="center" width="33%">10.2.12.198</th>
  </tr>
  <tr>
    <td colspan="2" align="center"><img src="docs/evidence/eks-pods-10-2-11-80.png" width="100%" alt="Pods on 10.2.11.80" /></td>
    <td colspan="2" align="center"><img src="docs/evidence/eks-pods-10-2-11-80-2.png" width="100%" alt="Pods on 10.2.11.80" /></td>
    <td colspan="2" align="center"><img src="docs/evidence/eks-pods-10-2-12-198.png" width="100%" alt="Pods on 10.2.12.198" /></td>
  </tr>
  <tr>
    <th colspan="2" align="center">10.2.12.82</th>
    <th colspan="2" align="center">10.2.13.167</th>
    <th colspan="2" align="center">10.2.13.184</th>
  </tr>
  <tr>
    <td colspan="2" align="center"><img src="docs/evidence/eks-pods-10-2-12-82.png" width="100%" alt="Pods on 10.2.12.82" /></td>
    <td colspan="2" align="center"><img src="docs/evidence/eks-pods-10-2-13-167.png" width="100%" alt="Pods on 10.2.13.167" /></td>
    <td colspan="2" align="center"><img src="docs/evidence/eks-pods-10-2-13-184.png" width="100%" alt="Pods on 10.2.13.184" /></td>
  </tr>
</table>

#### Topology spread

`check-az-spread.sh` checks that pods of the same service are spread across availability zones.

<table width="100%">
  <tr>
    <th align="center" width="50%">Command</th>
    <th align="center" width="50%">Result</th>
  </tr>
  <tr>
    <td align="center">
      <img src="docs/evidence/az-spread-check-script.png" width="100%" alt="AZ spread script" />
    </td>
    <td align="center">
      <img src="docs/evidence/az-spread-check-ok.png" width="100%" alt="AZ spread OK" />
    </td>
  </tr>
</table>

#### ECR `prod-*` repositories

One immutable repository per service, encrypted with AES-256.

<table width="100%">
  <tr>
    <td align="center">
      <img src="docs/evidence/ecr-prod-repositories.png" width="100%" alt="ECR" />
    </td>
  </tr>
</table>

---

## CI/CD

`.github/workflows/ci.yml` builds only the services that changed, pushes each image to ECR tagged with the git SHA, and writes that tag back into GitOps.

<table width="100%">
  <tr>
    <th colspan="2" align="center">Login</th>
  </tr>
  <tr>
    <td colspan="2" align="center">
      <img src="docs/evidence/github-actions-login.png" width="520" alt="GitHub login" />
    </td>
  </tr>
  <tr>
    <th align="center" width="50%">Running</th>
    <th align="center" width="50%">Success</th>
  </tr>
  <tr>
    <td align="center">
      <img src="docs/evidence/github-actions-ci-running.png" width="390" alt="CI running" />
    </td>
    <td align="center">
      <img src="docs/evidence/github-actions-ci-success.png" width="390" alt="CI success" />
    </td>
  </tr>
</table>

---

## GitOps (Argo CD)

Boutique workloads live in `gitops/boutique/`. Kustomize renders the storefront and operations Ingress. Kira is installed with `deploy-kira.sh`.

<table width="100%">
  <tr>
    <th colspan="2" align="center">Login</th>
  </tr>
  <tr>
    <td colspan="2" align="center">
      <img src="docs/evidence/argo-cd-login.png" width="520" alt="Argo login" />
    </td>
  </tr>
  <tr>
    <th align="center" width="50%">Synced</th>
    <th align="center" width="50%">Pods</th>
  </tr>
  <tr>
    <td align="center">
      <img src="docs/evidence/argo-cd-boutique-synced.png" width="390" alt="Synced" />
    </td>
    <td align="center">
      <img src="docs/evidence/argo-cd-boutique-pods.png" width="390" alt="Pods" />
    </td>
  </tr>
</table>

---

## Observability

| Signal | Tool | What it shows |
|---|---|---|
| **Metrics** | Prometheus → **Grafana** | CPU, memory, network for namespaces/pods |
| **Traces** | OpenTelemetry → **Jaeger** | Request paths across gateway and backends |
| **Logs** | Fluent Bit → **CloudWatch** | Per-service pod stdout/stderr streams and events |

### Metrics — Grafana

<table width="100%">
  <tr>
    <th colspan="3" align="center">Login</th>
  </tr>
  <tr>
    <td colspan="3" align="center">
      <img src="docs/evidence/grafana-login.png" width="520" alt="Grafana login" />
    </td>
  </tr>
  <tr>
    <th align="center" width="33%">Namespace CPU</th>
    <th align="center" width="33%">Pod memory</th>
    <th align="center" width="33%">Network bandwidth</th>
  </tr>
  <tr>
    <td align="center">
      <img src="docs/evidence/grafana-k8s-cpu-namespace.png" width="260" alt="CPU" />
    </td>
    <td align="center">
      <img src="docs/evidence/grafana-k8s-memory-usage.png" width="260" alt="Memory" />
    </td>
    <td align="center">
      <img src="docs/evidence/grafana-k8s-network-bandwidth.png" width="260" alt="Network" />
    </td>
  </tr>
</table>

### Traces — Jaeger

<table width="100%">
  <tr>
    <th colspan="2" align="center">Search</th>
  </tr>
  <tr>
    <td colspan="2" align="center">
      <img src="docs/evidence/jaeger-search.png" width="520" alt="Jaeger search" />
    </td>
  </tr>
  <tr>
    <th align="center" width="50%">Gateway traces</th>
    <th align="center" width="50%">API table</th>
  </tr>
  <tr>
    <td align="center">
      <img src="docs/evidence/jaeger-gateway-traces.png" width="390" alt="Gateway traces" />
    </td>
    <td align="center">
      <img src="docs/evidence/jaeger-api-traces-table.png" width="390" alt="API traces" />
    </td>
  </tr>
</table>

### Logs — CloudWatch

Fluent Bit shipped pod stdout and stderr to `/eks/boutique-prod/pods`. The same account view also shows the EKS control-plane log group, `/aws/lambda/aiops-fetch-*`, and the AgentCore harness runtime.

**Streams** is the filtered list for that service. **Events** is a sample from one stream. The samples include healthy gateway `/api` proxies, a clean user-service startup, and HTTP 200 responses from the frontend.

<table width="100%">
  <tr>
    <th colspan="2" align="center">Log groups (account overview)</th>
  </tr>
  <tr>
    <td colspan="2" align="center">
      <img src="docs/evidence/cloudwatch-log-groups.png" width="100%" alt="CloudWatch log groups" />
    </td>
  </tr>
  <tr>
    <th colspan="2" align="center"><code>auth</code> (:3002)</th>
  </tr>
  <tr>
    <th align="center" width="50%">Streams</th>
    <th align="center" width="50%">Events</th>
  </tr>
  <tr>
    <td align="center"><img src="docs/evidence/cloudwatch-auth-log-streams.png" width="390" alt="auth streams" /></td>
    <td align="center"><img src="docs/evidence/cloudwatch-auth-log-events.png" width="390" alt="auth events" /></td>
  </tr>
  <tr>
    <th colspan="2" align="center"><code>gateway</code> (:3001)</th>
  </tr>
  <tr>
    <th align="center" width="50%">Streams</th>
    <th align="center" width="50%">Events</th>
  </tr>
  <tr>
    <td align="center"><img src="docs/evidence/cloudwatch-gateway-log-streams.png" width="390" alt="gateway streams" /></td>
    <td align="center"><img src="docs/evidence/cloudwatch-gateway-log-events.png" width="390" alt="gateway events" /></td>
  </tr>
  <tr>
    <th colspan="2" align="center"><code>orders</code> (:3005)</th>
  </tr>
  <tr>
    <th align="center" width="50%">Streams</th>
    <th align="center" width="50%">Events</th>
  </tr>
  <tr>
    <td align="center"><img src="docs/evidence/cloudwatch-orders-log-streams.png" width="390" alt="orders streams" /></td>
    <td align="center"><img src="docs/evidence/cloudwatch-orders-log-events.png" width="390" alt="orders events" /></td>
  </tr>
  <tr>
    <th colspan="2" align="center"><code>order-service</code> (:3004)</th>
  </tr>
  <tr>
    <th align="center" width="50%">Streams</th>
    <th align="center" width="50%">Events</th>
  </tr>
  <tr>
    <td align="center"><img src="docs/evidence/cloudwatch-order-service-streams.png" width="390" alt="order-service streams" /></td>
    <td align="center"><img src="docs/evidence/cloudwatch-order-service-events.png" width="390" alt="order-service events" /></td>
  </tr>
  <tr>
    <th colspan="2" align="center"><code>product-service</code> (:3003)</th>
  </tr>
  <tr>
    <th align="center" width="50%">Streams</th>
    <th align="center" width="50%">Events</th>
  </tr>
  <tr>
    <td align="center"><img src="docs/evidence/cloudwatch-product-service-streams.png" width="390" alt="product streams" /></td>
    <td align="center"><img src="docs/evidence/cloudwatch-product-service-events.png" width="390" alt="product events" /></td>
  </tr>
  <tr>
    <th colspan="2" align="center"><code>user-service</code> (:3006)</th>
  </tr>
  <tr>
    <th align="center" width="50%">Streams</th>
    <th align="center" width="50%">Events</th>
  </tr>
  <tr>
    <td align="center"><img src="docs/evidence/cloudwatch-user-service-streams.png" width="390" alt="user streams" /></td>
    <td align="center"><img src="docs/evidence/cloudwatch-user-service-events.png" width="390" alt="user events" /></td>
  </tr>
  <tr>
    <th colspan="2" align="center"><code>frontend</code> (:3000)</th>
  </tr>
  <tr>
    <th align="center" width="50%">Streams</th>
    <th align="center" width="50%">Events</th>
  </tr>
  <tr>
    <td align="center"><img src="docs/evidence/cloudwatch-frontend-log-streams.png" width="390" alt="frontend streams" /></td>
    <td align="center"><img src="docs/evidence/cloudwatch-frontend-log-events.png" width="390" alt="frontend events" /></td>
  </tr>
</table>

---

## AIOps — Kira

The Streamlit UI `kira-ui`, in namespace `aiops`, uses IRSA to invoke harness **`kira_aiops`**. Gateway **`aiops-kira-tools`** routes each question to one Lambda:

| Lambda | Purpose | Network |
|---|---|---|
| `aiops-fetch-logs` | CloudWatch Logs | No VPC |
| `aiops-fetch-metrics` | Prometheus via NLB `prometheus-aiops` | VPC |
| `aiops-fetch-health` | EKS API + Prometheus | VPC |

`deploy-kira.sh` installs the UI. Validation used model `qwen.qwen3-32b-v1:0`. Tool paths are in `KIRA-FLOWS.txt`.

<table width="100%">
  <tr>
    <th colspan="2" align="center">Chat</th>
  </tr>
  <tr>
    <td colspan="2" align="center">
      <img src="docs/evidence/kira-aiops-chat.png" width="100%" alt="Kira chat" />
    </td>
  </tr>
  <tr>
    <th align="center" width="50%">Cluster health<br><code>aiops-fetch-health</code></th>
    <th align="center" width="50%">CPU / memory<br><code>aiops-fetch-metrics</code></th>
  </tr>
  <tr>
    <td align="center"><img src="docs/evidence/kira-eks-cluster-health.png" width="100%" alt="Cluster health" /></td>
    <td align="center"><img src="docs/evidence/kira-cpu-memory-query.png" width="100%" alt="CPU memory" /></td>
  </tr>
  <tr>
    <th align="center">CloudWatch errors<br><code>aiops-fetch-logs</code></th>
    <th align="center">Unhealthy deployment<br><code>aiops-fetch-health</code></th>
  </tr>
  <tr>
    <td align="center"><img src="docs/evidence/kira-cloudwatch-errors-query.png" width="100%" alt="CloudWatch errors" /></td>
    <td align="center"><img src="docs/evidence/kira-unhealthy-deployment.png" width="100%" alt="Unhealthy deployment" /></td>
  </tr>
</table>

<table width="100%">
  <tr>
    <th align="center">Lambda functions</th>
  </tr>
  <tr>
    <td align="center">
      <img src="docs/evidence/lambda-aiops-functions.png" width="100%" alt="Lambda functions" />
    </td>
  </tr>
</table>

---

## Security & networking

- EKS nodes are private. Public entry is only through the load balancer, with ACM certificates.
- The storefront ALB group `boutique-app` has a WAF web ACL. Argo CD, Grafana, Jaeger, and Kira share `boutique-ops`.
- IRSA covers the load balancer controller, external-dns, Fluent Bit, External Secrets, Karpenter, and `kira-ui`.
- Kira calls Bedrock with `InvokeHarness` and `InvokeAgentRuntime` through that role. The UI does not mount long-lived AWS keys.
- The database password is in Secrets Manager (`boutique/prod/postgres`) and is synced by External Secrets into `boutique-secrets`.

---

## Operate

| Action | Runbook |
|---|---|
| Restore | `RESTORATION.txt` in the private repository |
| Tear down | `TEARDOWN.txt` in the private repository |

---

## Author

**Olivier Sims** (`oliversims`) — cloud-native DevOps and AIOps portfolio on AWS. Built and validated in production, then decommissioned for cost control. The private codebase and the evidence above are what remain.
