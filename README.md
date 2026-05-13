# 📘 DevOps Portfolio - Project Overview

<p align="center">
  <a href="https://github.com/melan-devops1/portfolio-app"><img src="https://img.shields.io/badge/App-Spring_Boot_3.5-6DB33F?logo=springboot&logoColor=white"/></a>
  <a href="https://github.com/melan-devops1/portfolio-infra"><img src="https://img.shields.io/badge/Infra-Terraform_1.14-7B42BC?logo=terraform&logoColor=white"/></a>
  <a href="https://github.com/melan-devops1/portfolio-manifests"><img src="https://img.shields.io/badge/GitOps-ArgoCD-EF7B4D?logo=argo&logoColor=white"/></a>
  <img src="https://img.shields.io/badge/Cloud-AWS-232F3E?logo=amazonwebservices&logoColor=white"/>
  <img src="https://img.shields.io/badge/Orchestration-Kubernetes_1.33-326CE5?logo=kubernetes&logoColor=white"/>
</p>

---

## 🎯 프로젝트 목표

이커머스 플랫폼을 가정한 마이크로서비스를 **AWS EKS 위에서 운영하는 실전형 DevOps 플랫폼**을
처음부터 끝까지 혼자 구축한 1인 프로젝트입니다.

핵심 질문은 다음과 같았습니다:
- 🔹 인프라를 코드로(IaC) 완전히 재현 가능하게 만들 수 있는가?
- 🔹 개발자가 `git push` 한 번으로 프로덕션까지 배포되는 파이프라인을 만들 수 있는가?
- 🔹 장애가 발생했을 때 **메트릭·로그·트레이스** 세 축으로 원인을 추적할 수 있는가?
- 🔹 SLA 99.9%를 지키기 위한 알림·에스컬레이션 체계를 수립할 수 있는가?

---

## 📦 Repository Map

| Repository | 설명 |
|---|---|
| **[portfolio-app](https://github.com/melan-devops1/portfolio-app)** | Spring Boot 3.5.13 + Java 21 마이크로서비스 3개 (product/order/payment). Multi-stage Dockerfile, OpenTelemetry Java Agent 동봉, GitHub Actions CI 포함 |
| **[portfolio-infra](https://github.com/melan-devops1/portfolio-infra)** | Terraform 모듈 기반 AWS 인프라 (VPC, EKS, ECR, RDS, ALB Controller IAM, EBS CSI IAM). S3 backend + native locking (use_lockfile) |
| **[portfolio-manifests](https://github.com/melan-devops1/portfolio-manifests)** | ArgoCD가 watch하는 GitOps 레포. 자체 앱은 Kustomize, 3rd party는 Helm. monitoring/logging/tracing 스택 포함 |

---

## 🚦 Project Status

> Phase별 진행 현황. 1인 프로젝트 마무리 시점 기준.

- [x] **Phase 0**: 기반 세팅 (Org, 4 repos, AWS 계정, Budgets $80)
- [x] **Phase 1**: Spring 앱 + 컨테이너화 (3 services, Trivy baseline 통과, 130MB)
- [x] **Phase 2**: Terraform AWS 인프라 (VPC, EKS 1.33, ECR, GitHub OIDC)
- [x] **Phase 3**: K8s 배포 + CI/CD + ArgoCD GitOps
- [x] **Phase 4**: Observability 풀스택 (kube-prometheus-stack, EFK, Jaeger, RDS, OpenTelemetry)
- [ ] **Phase 5+ (보류)**: Istio service mesh, Renovate/Dependabot, PagerDuty 에스컬레이션, 30일 SLO 실측

> Phase 5 이후는 다른 우선순위 작업으로 인해 1인 프로젝트는 여기서 일단락. 미구현 항목은 위 마지막 항목 참조.

---

## 🏗 Architecture

### 1. End-to-End Flow — 코드부터 배포까지

```
  [Developer]
      │ git push
      ▼
  GitHub: portfolio-app ──────▶ GitHub Actions
                                │  pre-checks.yaml     (PR open/sync)
                                │   · paths-filter + matrix lint+test
                                │  build-and-push.yaml (main merge)
                                │   · OIDC → AWS
                                │   · BuildKit (SBOM + provenance)
                                │   · Trivy CRITICAL gate
                                │   ├─── docker push ──────▶ AWS ECR
                                │   │                       (IMMUTABLE)
                                │   └─── kustomize edit set image
                                ▼
                                GitHub: portfolio-manifests
                                apps/<svc>/base/kustomization.yaml
                                 newTag = <commit-sha>
                                     │ watch
                                     ▼
                                ArgoCD  (ns: argocd)
                                 auto-sync + selfHeal + prune
                                     │ apply manifests
                                     ▼
  ┌─────────────── AWS VPC 10.0.0.0/16 (ap-northeast-2) ──────────────────────┐
  │                                                                           │
  │   ┌───────────── AWS Resources (EKS 외부) ──────────────────┐              │
  │   │  · ALB(internet-facing) ×5 — group별 분리               │              │
  │   │      ⤷ portfolio / argocd / grafana / kibana / jaeger  │              │
  │   │  · RDS PostgreSQL 15.17 db.t3.micro (intra subnet,     │              │
  │   │     5432 inbound: EKS 노드 SG만)                        │              │
  │   │  · NAT GW (AZ-a) — Pod의 outbound 인터넷                 │              │
  │   └────┬─────────────────────┬───────────────────┬─────────┘              │
  │        │ ① ALB→Pod IP        │ ② JDBC 5432       │ ③ NAT egress         │
  │        │   (target-type=ip)  │                   │   (ECR pull,           │
  │        │                     │                   │    Slack webhook,      │
  │        │                     │                   │    ArgoCD git)         │
  │   ┌────▼─────────────────────┼───────────────────┼───────────────────┐    │
  │   │           EKS 1.33 Cluster (Pods in private subnets)             │    │
  │   │                          │                   │                   │    │
  │   │  ─── ns: default ────────┼───────────────────┼─────────────┐     │    │
  │   │  │  Ingress(alb) "portfolio-ingress":        │             │     │    │
  │   │  │   /api/products → Service product-service:8081          │     │    │
  │   │  │   /api/orders   → Service order-service:8082            │     │    │
  │   │  │   /api/payments → Service payment-service:8083 (Chaos)  │     │    │
  │   │  │                          │                              │     │    │
  │   │  │  Pods (3 Deployments + 3 HPA):                          │     │    │
  │   │  │   · order ─▶ product (8081)   ─▶ payment (8083)         │     │    │
  │   │  │   · SPRING_PROFILES_ACTIVE=prod                         │     │    │
  │   │  │   · envFrom: portfolio-db-config / -credentials ─▶ ②   │     │    │
  │   │  │   · JAVA_TOOL_OPTIONS=-javaagent:/opt/otel/...          │     │    │
  │   │  │   · OTLP endpoint=jaeger.tracing.svc.cluster.local:4317 │     │    │
  │   │  │      └─────────────────────────────┐                    │     │    │
  │   │  └──────────────────────────────────  │  ──────────────────┘     │    │
  │   │                                       │ ④ OTLP gRPC (cluster 내) │    │
  │   │                                       ▼                          │    │
  │   │  ─── ns: tracing ──────────────────────────────────────┐         │    │
  │   │  │  Service "jaeger" → Pod (all-in-one, memory storage)│         │    │
  │   │  │   포트 4317(OTLP gRPC) / 4318(HTTP) / 16686(Query UI)│         │    │
  │   │  └──────────────────────────────────────────────────────┘        │    │
  │   │                                                                  │    │
  │   │  ─── ns: monitoring (kube-prometheus-stack v84.3.0) ────────┐    │    │
  │   │  │  Prometheus ◀─── scrape /actuator/prometheus from default│    │    │
  │   │  │            │     (ServiceMonitor × 3 in monitoring ns,   │    │    │
  │   │  │            │      namespaceSelector: default)            │    │    │
  │   │  │            ▼                                             │    │    │
  │   │  │  PrometheusRule (4 alerts: HighErrorRate /               │    │    │
  │   │  │     HighLatencyP99 / PodCrashLoopBackOff / PodNotReady)  │    │    │
  │   │  │            │                                             │    │    │
  │   │  │            ▼ firing                                      │    │    │
  │   │  │  Alertmanager ──▶ Slack webhook ──▶ ③ NAT ──▶ #ops-alert │    │    │
  │   │  │     (route 설정: AlertmanagerConfig CRD;                  │    │    │
  │   │  │      webhook URL: Secret "alertmanager-slack-webhook")   │    │    │
  │   │  │                                                          │    │    │
  │   │  │  Grafana (PVC gp3, ALB Ingress "grafana")                │    │    │
  │   │  └──────────────────────────────────────────────────────────┘    │    │
  │   │                                                                  │    │
  │   │  ─── ns: logging ──────────────────────────────────────────┐     │    │
  │   │  │  Fluent Bit DaemonSet — 모든 노드에서                     │     │    │
  │   │  │   tail /var/log/containers/*.log → kubernetes filter    │     │    │
  │   │  │              │ Logstash_Prefix portfolio                │     │    │
  │   │  │              ▼ (cluster 내 9200)                        │     │    │
  │   │  │  Service "elasticsearch":9200 → Pod (single node)       │     │    │
  │   │  │  Kibana (ALB Ingress "kibana")                          │     │    │
  │   │  └─────────────────────────────────────────────────────────┘     │    │
  │   │                                                                  │    │
  │   │  ─── ns: argocd ───────────────────────────────────────────┐     │    │
  │   │  │  ArgoCD v3.3.6 (Helm chart 9.5.0)                       │     │    │
  │   │  │   · 4개 Application (자세히는 §3)                         │     │    │
  │   │  │   · GitHub repo poll (default 3분, ③ NAT 경유)           │     │    │
  │   │  │  ArgoCD UI Ingress(alb) "argocd"                         │    │    │
  │   │  └──────────────────────────────────────────────────────────┘    │    │
  │   │                                                                  │    │
  │   │  ─── ns: kube-system ─────────────────────────────────────┐      │    │
  │   │  │  AWS Load Balancer Controller (helm v1.14.0)           │      │    │
  │   │  │   — Ingress 자원 watch → AWS ALB 생성/관리               │      │    │
  │   │  │  metrics-server (helm) — HPA × 3가 의존                 │      │    │
  │   │  │  EBS CSI Driver (helm) — Prometheus PVC gp3 의존        │      │    │
  │   │  │  CoreDNS / kube-proxy / vpc-cni / pod-identity-agent   │      │    │
  │   │  │  (Terraform addons로 자동 설치)                          │      │    │
  │   │  └────────────────────────────────────────────────────────┘      │    │
  │   │                                                                  │    │
  │   │  · · · · · · · · · · · · · · · · · · · · · · · · · · · · · ·     │    │
  │   │  : Istio Service Mesh (계획 단계, 미구현 — Phase 5+)         :      │    │
  │   │  · · · · · · · · · · · · · · · · · · · · · · · · · · · · · ·     │    │
  │   └──────────────────────────────────────────────────────────────────┘    │
  └───────────────────────────────────────────────────────────────────────────┘

   외부:  Internet  ──▶ ALB(internet-facing) (사용자 API/UI 접근, ①)
          ECR public endpoint  ◀── NAT GW egress (Pod image pull, ③)
          Slack webhook        ◀── NAT GW egress (③)
          GitHub               ◀── NAT GW egress (ArgoCD git pull, ③)
```

> 점선은 계획 단계로만 남아있는 컴포넌트.
> VPC endpoint는 미구성 — ECR / Slack / GitHub 호출은 모두 NAT 경유 egress.

### 2. VPC 인프라 수준 아키텍처

`portfolio-infra/modules/vpc`로 직접 작성. 2-AZ × 3-tier subnet, Single NAT (비용 절감).

```
                              ┌──────────────────────────┐
              Internet ◀─────▶│ Internet Gateway (IGW)   │
                              └────────────┬─────────────┘
                                           │
                              ┌────────────┴─────────────────────────────┐
                              │ AWS Load Balancer Controller가 5개 ALB    │
                              │ 생성 (모두 internet-facing, target=ip):    │
                              │  · group.name=portfolio  (앱 API)         │
                              │  · group.name=argocd     (ArgoCD UI)     │
                              │  · group.name=grafana    (Grafana UI)    │
                              │  · group.name=kibana     (Kibana UI)     │
                              │  · group.name=jaeger     (Jaeger UI)     │
                              │ 각 ALB는 양 AZ public subnet에 ENI 배치    │
                              └─────┬──────────────────────────┬─────────┘
                                    │ (ENI)                    │ (ENI)
   ┌────────────────────────────────┼──────────────────────────┼──────────────────┐
   │            VPC  10.0.0.0/16  (portfolio-dev)              │                  │
   │                                │                          │                  │
   │   ╔═══ AZ-a (ap-northeast-2a) ═╪══╗  ╔════════════════════╪═AZ-c════════╗    │
   │   ║                            │  ║  ║                    │             ║    │   
   │   ║ ┌─ Public 10.0.0.0/24 ───┐ │  ║  ║ ┌─ Public 10.0.1.0/24 ─┐         ║    │
   │   ║ │  EIP + NAT GW          │◀┘  ║  ║ │ (NAT GW 없음 — Single │         ║    │
   │   ║ │  ALB ENI × 5           │    ║  ║ │  NAT, AZ-a 공유)      │         ║    │
   │   ║ │                        │    ║  ║ │  ALB ENI × 5          │        ║    │
   │   ║ └────────▲───────────────┘    ║  ║ └───────────────────────┘        ║    │
   │   ║          │ 0.0.0.0/0 egress   ║  ║                                  ║    │
   │   ║          │ (Private RT가 NAT GW-a로 라우팅, single NAT 공유)           ║    │
   │   ║ ┌─ Private 10.0.16.0/20 ──┐   ║  ║ ┌─ Private 10.0.32.0/20 ─────┐    ║    │
   │   ║ │ EKS Worker (t3.large)   │   ║  ║ │ EKS Worker (t3.large)      │    ║    │
   │   ║ │  · Pods (default,       │   ║  ║ │  · Pods (default,          │    ║    │
   │   ║ │    monitoring, logging, │   ║  ║ │    monitoring, logging,    │    ║    │
   │   ║ │    tracing, argocd ns)  │   ║  ║ │    tracing, argocd ns)     │    ║    │
   │   ║ │ EKS Control Plane ENI   │   ║  ║ │ EKS Control Plane ENI      │    ║    │
   │   ║ └─────────────────────────┘   ║  ║ └────────────────────────────┘    ║    │
   │   ║                               ║  ║                                   ║    │
   │   ║ ┌─ Intra 10.0.48.0/24 ────┐   ║  ║ ┌─ Intra 10.0.49.0/24 ───────┐    ║    │
   │   ║ │ (0.0.0.0/0 라우트 없음)   │   ║  ║ │ (0.0.0.0/0 라우트 없음)      │    ║    │
   │   ║ │                         │   ║  ║ │                            │    ║    │
   │   ║ │  RDS PostgreSQL 15.17   │   ║  ║ │  (multi_az=false → 인스턴스 │     ║    │
   │   ║ │  db.t3.micro            │   ║  ║ │   없음. DB Subnet Group에   │     ║    │
   │   ║ │  single-AZ instance     │   ║  ║ │   양 AZ 모두 등록)           │     ║   │
   │   ║ │  5432 ◀── EKS 노드 SG만  │   ║  ║ │                            │     ║   │
   │   ║ └─────────────────────────┘   ║  ║ └────────────────────────────┘     ║   │
   │   ╚═══════════════════════════════╝  ╚════════════════════════════════════╝   │
   │                                                                               │
   └───────────────────────────────────────────────────────────────────────────────┘

Subnet Tier 차이:
  - Public  : IGW 라우트(0.0.0.0/0 → IGW), ALB·NAT 위치
  - Private : NAT 라우트(0.0.0.0/0 → NAT GW-a, single NAT 공유), EKS 워커·Pod
  - Intra   : 인터넷 라우트 없음, RDS·내부 격리 자산 (EKS 노드 SG에서만 5432 허용)

Pod IP 소진 대비: Private = /20 (4091 IP), Public/Intra = /24 (251 IP).
RDS 보안 그룹: 5432 inbound는 EKS 노드 SG에서만 허용 (modules/rds/main.tf 정합).
Single NAT: AZ-a의 NAT GW를 양 AZ의 Private 서브넷이 공유 (envs/dev: single_nat_gateway=true).
```

### 3. ArgoCD 구조 — 4개 Application

ArgoCD는 `argocd` namespace에 설치 (Helm chart `argo/argo-cd` v9.5.0). 4개 Application으로 클러스터 전체 상태 관리.

```
┌──────────────────────────────────────────────────────────────────────┐
│                    Git (portfolio-manifests, main)                   │
│                                                                      │
│  apps/all/overlays/dev/        ← Kustomize (in-repo manifests)       │
│  monitoring/.../values.yaml    │                                     │
│  logging/.../values.yaml       │ ref: values  (Helm + Git 결합)       │
│  tracing/values.yaml           │                                     │
└──────────┬──────────────┬──────────────┬──────────────┬──────────────┘
           │              │              │              │
           │ (Kustomize)  │ (Helm+values)│ (Helm+values)│ (Helm+values)
           ▼              ▼              ▼              ▼
   ┌───────────────────────────────────────────────────────────────┐
   │              ArgoCD (ns: argocd, v3.3.6 / chart 9.5.0)        │
   │                                                               │
   │  Application: portfolio-app                                   │
   │   · source: portfolio-manifests / apps/all/overlays/dev       │
   │   · sync: auto + selfHeal + prune                             │
   │   · ignoreDifferences: Deployment.spec.replicas (HPA 충돌 방지) │
   │   · destination: in-cluster, ns=default                       │
   │                                                               │
   │  Application: kube-prometheus-stack                           │
   │   · source A: helm prometheus-community/kube-prometheus-stack │
   │               v84.3.0                                         │
   │   · source B: portfolio-manifests / monitoring/.../values.yaml│
   │   · sync: auto + selfHeal,  CreateNamespace=true              │
   │   · destination: in-cluster, ns=monitoring                    │
   │                                                               │
   │  Application: fluent-bit                                      │
   │   · source A: helm fluent/fluent-bit  v0.57.3                 │
   │   · source B: portfolio-manifests / logging/fluent-bit/values │
   │   · sync: auto + selfHeal,  DaemonSet                         │
   │   · destination: in-cluster, ns=logging                       │
   │                                                               │
   │  Application: jaeger                                          │
   │   · source A: helm jaegertracing/jaeger  v4.7.0               │
   │   · source B: portfolio-manifests / tracing/values.yaml       │
   │   · sync: auto + selfHeal,  all-in-one + memory storage       │
   │   · destination: in-cluster, ns=tracing                       │
   └────┬──────────────┬───────────────┬───────────────┬───────────┘
        │ apply        │ apply         │ apply         │ apply
        ▼              ▼               ▼               ▼
  ┌──────────┐  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
  │ ns:      │  │ ns:          │ │ ns:          │ │ ns:          │
  │ default  │  │ monitoring   │ │ logging      │ │ tracing      │
  │          │  │              │ │              │ │              │
  │ product  │  │ Prometheus   │ │ Fluent Bit   │ │ Jaeger       │
  │ order    │  │ Grafana      │ │ (DaemonSet)  │ │ (all-in-one) │
  │ payment  │  │ Alertmanager │ │              │ │              │
  │ + HPA×3  │  │              │ │              │ │              │
  └──────────┘  └──────────────┘ └──────────────┘ └──────────────┘

ArgoCD가 관리하지 않는 자원 (kubectl 또는 helm으로 직접 배포):

  [클러스터 부트스트랩 — ArgoCD 의존성/전제]
  · AWS Load Balancer Controller    helm v1.14.0 (kube-system ns)
                                    → 5개 Ingress가 작동하기 위한 전제
  · metrics-server                  helm (kube-system ns)
                                    → HPA × 3가 의존 (CPU resource metric)
  · EBS CSI Driver                  helm (kube-system ns)
                                    → Prometheus PVC(gp3)가 의존
                                    (terraform addons에 미포함)
  · ArgoCD 자체                     helm chart 9.5.0 (argocd ns)
                                    chicken-and-egg

  [5개 Ingress — 각 다른 group.name → 5개 ALB 생성]
  · infrastructure/ingress/         앱 API     (group.name=portfolio)
  · argocd/ingress.yaml             ArgoCD UI  (group.name=argocd)
  · monitoring/grafana-ingress      Grafana UI (group.name=grafana)
  · logging/kibana-ingress          Kibana UI  (group.name=kibana)
  · tracing/jaeger-ingress          Jaeger UI  (group.name=jaeger)

  [Stateful / 직접 YAML 자원]
  · Elasticsearch + Kibana          logging/elasticsearch.yaml,
                                    logging/kibana.yaml
  · logging namespace               logging/namespace.yaml
                                    (Helm chart들은 CreateNamespace=true)
  · ServiceMonitor × 3,             monitoring/service-monitors/*.yaml,
    PrometheusRule,                 monitoring/prometheus-rules.yaml,
    AlertmanagerConfig              monitoring/alertmanager-config.yaml

  [Secret / ConfigMap — Terraform output → kubectl로 외부 주입]
  · portfolio-db-config (ConfigMap)  Terraform output: rds_jdbc_url
  · portfolio-db-credentials (Secret) Terraform output: rds_username,
                                                       rds_password
  · alertmanager-slack-webhook (Secret) Slack 워크스페이스의 webhook URL
                                        — AlertmanagerConfig가 참조
    (chicken-and-egg 회피, ADR-0023)
```

---

## 🛠 Tech Stack

### Application
- **Spring Boot** 3.5.13, **Java** 21 LTS (Eclipse Temurin)
- **DB**: H2 in-memory (local) / PostgreSQL 15 RDS (운영)
- **HTTP**: RestClient (RestTemplate 미사용)
- **Logging**: logstash-encoder 8.0 (JSON 구조화)
- **에러 응답**: RFC 7807 ProblemDetail 표준

### Infrastructure & Cloud
- **AWS**: EKS, VPC, ECR, RDS, IAM, S3 (state)
- **IaC**: Terraform 1.14 + AWS Provider ~> 6.0
- **EKS 모듈**: `terraform-aws-modules/eks/aws ~> 21.0`
- **EBS CSI / ALB Controller IAM**: 자체 Pod Identity 모듈

### Orchestration & Deployment
- **Kubernetes**: 1.33 (AL2023 노드, t3.large × 2, HPA로 최대 4)
- **권한 모델**: EKS Pod Identity (IRSA 미사용 — 2026 표준)
- **Packaging**: Kustomize (자체 앱) + Helm (3rd party 차트)
- **CI**: GitHub Actions + OIDC 페더레이션 (정적 자격증명 0개)
- **CD / GitOps**: ArgoCD v3.3.6 (Helm chart 9.5.0, auto-sync + self-heal)

### Observability (3-Pillar)
- **Metrics**: Prometheus + Grafana (kube-prometheus-stack)
- **Logs**: Elasticsearch + Fluent Bit + Kibana (Fluentd 미선택, ADR-0029)
- **Traces**: Jaeger + OpenTelemetry Java Agent (v2.26.1)
- **Alerts**: Alertmanager → Slack (#ops-alert, severity별 라우팅)

### Network & Security
- **Ingress**: AWS Load Balancer Controller (ALB)
- **Image Scan**: Trivy CI 차단 (CRITICAL) + ECR scan_on_push (다층 방어)
- **Image 정책**: IMMUTABLE 태그, untagged 1일/tagged 10개 lifecycle
- **Supply Chain**: Docker BuildKit attestation + SBOM 자동 첨부

---

## 📊 Key Achievements

운영 수치를 측정 가능한 형태로 박제.

- ✅ **Docker 이미지 사이즈**: 130MB (순진한 빌드 266MB 대비 **51% 감소**)
- ✅ **Trivy 보안 스캔**: 3개 이미지 모두 CRITICAL **0개** (HIGH 2개는 영향도 평가 후 baseline 등록)
- ✅ **Terraform 자원**: VPC + EKS + ECR + RDS + ALB/EBS Pod Identity IAM
- ✅ **인프라 apply 시간**: ~13분 (EKS Control Plane 생성 ~9분이 가장 오래)
- ✅ **K8s Pod 부팅**: ~50초 (Spring Boot 3.5.13 + Alpine JRE)
- ✅ **AWS 운영비**: 매일 destroy 사이클 운영 → 월 $30~40 (Budgets $80 알림)
- ✅ **CI 자동화**: paths-filter로 변경 서비스만 matrix 빌드, idempotent ECR push, manifests 자동 갱신 → ArgoCD가 감지해 자동 배포
- ✅ **SLO 박제**: 가용성 99.9% / P99 < 2초 / 5xx < 5% — PromQL + Alertmanager 라우팅까지 구현 ([sla.md](https://github.com/melan-devops1/portfolio-infra/blob/main/docs/sla.md))

### 미측정 (Phase 5+)
- 🎯 30일 rolling SLO 실측치 (매일 destroy 사이클이라 누적 데이터 없음)
- 🎯 배포 리드 타임 (commit → prod) 실측치

---

## 📝 Architecture Decision Records

운영 환경에서 내린 기술 선택을 ADR로 박제. 총 29개 (앱 11 + 인프라 18).

### App-tier (portfolio-app/docs/adr/)
| | 결정 |
|---|---|
| [0001](https://github.com/melan-devops1/portfolio-app/blob/main/docs/adr/0001-microservice-structure.md) | 마이크로서비스 분리 구조 (product/order/payment 단방향) |
| [0002](https://github.com/melan-devops1/portfolio-app/blob/main/docs/adr/0002-gradle-git-properties.md) | gradle-git-properties로 commit hash 자동 노출 |
| [0003](https://github.com/melan-devops1/portfolio-app/blob/main/docs/adr/0003-structured-logging.md) | logstash-encoder 8.0 (Spring Boot 내장 대신) |
| [0004](https://github.com/melan-devops1/portfolio-app/blob/main/docs/adr/0004-problem-detail.md) | RFC 7807 ProblemDetail 표준 에러 응답 |
| [0005](https://github.com/melan-devops1/portfolio-app/blob/main/docs/adr/0005-distributed-tracing.md) | MDC + X-Request-Id 분산 추적 기반 |
| [0006](https://github.com/melan-devops1/portfolio-app/blob/main/docs/adr/0006-dependency-version-unification.md) | 전 서비스 의존성 버전 통일 |
| [0007](https://github.com/melan-devops1/portfolio-app/blob/main/docs/adr/0007-chaos-simulation.md) | payment-service 의도적 Chaos (5% 에러 + 100~2000ms 지연) |
| [0008](https://github.com/melan-devops1/portfolio-app/blob/main/docs/adr/0008-docker-build-strategy.md) | 호스트 빌드 → Docker 패키징 분리 |
| [0009](https://github.com/melan-devops1/portfolio-app/blob/main/docs/adr/0009-docker-image-optimization.md) | Multi-stage + Layered + Alpine JRE (130MB) |
| [0010](https://github.com/melan-devops1/portfolio-app/blob/main/docs/adr/0010-security-scan-baseline.md) | Trivy CRITICAL 차단 / HIGH baseline 정책 |
| [0020](https://github.com/melan-devops1/portfolio-app/blob/main/docs/adr/0020-ci-pipeline-github-actions.md) | GitHub Actions CI (OIDC + paths-filter + idempotent ECR push) |

### Infra-tier (portfolio-infra/docs/adr/)
| | 결정 |
|---|---|
| [0011](https://github.com/melan-devops1/portfolio-infra/blob/main/docs/adr/0011-terraform-state-backend.md) | S3 backend (versioning + AES256), bootstrap만 로컬 state |
| [0012](https://github.com/melan-devops1/portfolio-infra/blob/main/docs/adr/0012-vpc-design.md) | VPC 직접 작성 + Single NAT + 3-tier subnet |
| [0013](https://github.com/melan-devops1/portfolio-infra/blob/main/docs/adr/0013-s3-native-locking.md) | DynamoDB lock → S3 native locking (`use_lockfile`) |
| [0014](https://github.com/melan-devops1/portfolio-infra/blob/main/docs/adr/0014-eks-official-module.md) | EKS는 공식 모듈 wrapping (~> 21.0) |
| [0015](https://github.com/melan-devops1/portfolio-infra/blob/main/docs/adr/0015-pod-identity-over-irsa.md) | Pod Identity 채택 (IRSA supersede) |
| [0016](https://github.com/melan-devops1/portfolio-infra/blob/main/docs/adr/0016-ecr-strategy.md) | ECR 서비스별 분리 + IMMUTABLE + scan_on_push + lifecycle |
| [0017](https://github.com/melan-devops1/portfolio-infra/blob/main/docs/adr/0017-github-actions-oidc.md) | OIDC + Role 분리, 정적 자격증명 0개 |
| [0018](https://github.com/melan-devops1/portfolio-infra/blob/main/docs/adr/0018-kustomize-over-helm.md) | (앱 manifest) Kustomize 채택 |
| [0019](https://github.com/melan-devops1/portfolio-infra/blob/main/docs/adr/0019-alb-ingress.md) | AWS Load Balancer Controller + ALB Ingress |
| [0021](https://github.com/melan-devops1/portfolio-infra/blob/main/docs/adr/0021-argocd-gitops.md) | ArgoCD GitOps (auto-sync + self-heal + ignoreDifferences) |
| [0022](https://github.com/melan-devops1/portfolio-infra/blob/main/docs/adr/0022-observability-metrics.md) | kube-prometheus-stack + ServiceMonitor + PrometheusRule |
| [0023](https://github.com/melan-devops1/portfolio-infra/blob/main/docs/adr/0023-rds-postgresql-and-configmap-secret.md) | RDS PostgreSQL 15 + ConfigMap/Secret 패턴 |
| [0024](https://github.com/melan-devops1/portfolio-infra/blob/main/docs/adr/0024-efk-logging-stack.md) | EFK Stack (Elasticsearch + Fluent Bit + Kibana) |
| [0025](https://github.com/melan-devops1/portfolio-infra/blob/main/docs/adr/0025-distributed-tracing-jaeger.md) | Jaeger + OpenTelemetry Java Agent (traces only) |
| [0026](https://github.com/melan-devops1/portfolio-infra/blob/main/docs/adr/0026-alertmanager-matcher-strategy.md) | AlertmanagerConfig matcherStrategy=None |
| [0027](https://github.com/melan-devops1/portfolio-infra/blob/main/docs/adr/0027-manifest-application-strategy.md) | Manifest 적용 전략 (Helm/kubectl/ArgoCD 경계) |
| [0028](https://github.com/melan-devops1/portfolio-infra/blob/main/docs/adr/0028-log-platform-elasticsearch-over-splunk.md) | 로그 플랫폼 Elasticsearch 선택 (vs Splunk) |
| [0029](https://github.com/melan-devops1/portfolio-infra/blob/main/docs/adr/0029-log-pipeline-fluent-bit-over-fluentd-logstash.md) | 로그 파이프라인 Fluent Bit 선택 (vs Fluentd/Logstash) |

---

## 🔧 운영 문서

Phase 4 마무리 시점에 박제한 운영 표준:

- [sla.md](https://github.com/melan-devops1/portfolio-infra/blob/main/docs/sla.md) — SLO/SLI 정의, 에러 예산 운용, 위반 대응 5단계
- [metrics-spec.md](https://github.com/melan-devops1/portfolio-infra/blob/main/docs/metrics-spec.md) — RED 메트릭, PromQL, 알림 임계치 근거
- [incident-report.md](https://github.com/melan-devops1/portfolio-infra/blob/main/docs/incident-report.md) — 사고 보고 양식
- [docs/runbook/](https://github.com/melan-devops1/portfolio-infra/tree/main/docs/runbook) — 502 cascade / Alertmanager Slack 미발화 / HPA boot race 3개 Runbook
- [docs/kibana/](https://github.com/melan-devops1/portfolio-infra/tree/main/docs/kibana) — SLA 대시보드 saved object

---

## 🎬 Screenshots

> 🚧 Phase 4 Observability 도입은 완료. 스크린샷 박제는 별도 작업으로 남겨둠.

---

## 📬 Contact

- 📧 Email: melanie0617@gmail.com
- 📝 Blog: [Notion Blog](https://0617.notion.site/Melanie-s-Devlog-3252455b985d4c0584eb9a94e54a6416?source=copy_link)
