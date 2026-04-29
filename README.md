# 📘 DevOps Portfolio - Project Overview

<!-- TODO(Phase 6): 아키텍처 다이어그램 이미지 추가
<p align="center">
  <img src="./docs/diagrams/architecture.png" alt="Architecture" width="800"/>
</p>
-->

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
| **[portfolio-app](https://github.com/melan-devops1/portfolio-app)** | Spring Boot 3.5.13 + Java 21 마이크로서비스 3개 (product/order/payment). Multi-stage Dockerfile, 로컬 docker-compose 포함 |
| **[portfolio-infra](https://github.com/melan-devops1/portfolio-infra)** | Terraform 모듈 기반 AWS 인프라 (VPC, EKS, ECR, IAM). S3 backend + native locking (use_lockfile) |
| **[portfolio-manifests](https://github.com/melan-devops1/portfolio-manifests)** | ArgoCD가 watch하는 GitOps 레포. Kustomize base + overlays 구조, 3rd party 차트는 Helm |

---

## 🚦 Project Status

> Phase별 진행 현황. 자세한 트래킹은 [PROJECT_CONTEXT.md](./PROJECT_CONTEXT.md) 참조.

- [x] **Phase 0**: 기반 세팅 (Org, 4 repos, AWS 계정, Budgets $80)
- [x] **Phase 1**: Spring 앱 + 컨테이너화 (3 services, Trivy baseline 통과)
- [x] **Phase 2**: Terraform AWS 인프라 (VPC, EKS 1.33, ECR, GitHub OIDC)
- [ ] **Phase 3**: K8s 배포 + CI/CD (현재 진행 중 — product-service 배포 검증 완료)
- [ ] **Phase 4**: Observability 풀스택 (Prometheus, EFK, Jaeger)
- [ ] **Phase 5**: Service Mesh (Istio mTLS, Canary)
- [ ] **Phase 6**: 문서화 + 블로그 포스팅

---

## 🏗 Architecture

### End-to-End Flow

```
┌─────────────┐       ┌──────────────────┐      ┌──────────────────┐
│  Developer  │─────▶│  GitHub (App)    │─────▶│ GitHub Actions   │
└─────────────┘       └──────────────────┘      │  - Build         │
                                                │  - Test          │
                                                │  - Trivy Scan    │
                                                │  - Push to ECR   │
                                                │  - Bump Manifest │
                                                └────────┬─────────┘
                                                         │
                                                         ▼
                                                ┌──────────────────┐
                                                │ GitHub(Manifests)│
                                                └────────┬─────────┘
                                                         │ watch
                                                         ▼
┌──────────────────────────────────────────────┐   ┌──────────────┐
│              AWS EKS 1.33 Cluster            │◀──│    ArgoCD    │
│                                              │   └──────────────┘
│  ┌────────────────────────────────────────┐  │
│  │         Istio Service Mesh             │  │
│  │  ┌───────┐   ┌──────┐   ┌──────────┐   │  │
│  │  │product│◀──│order │──▶│ payment  │   │  │
│  │  └───────┘   └──────┘   └──────────┘   │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  ┌──────────────┐ ┌──────────────┐           │
│  │  Prometheus  │ │  EFK Stack   │           │
│  │  + Grafana   │ │  (Fluent Bit)│           │
│  └──────────────┘ └──────────────┘           │
│                                              │
│  ┌──────────────┐ ┌──────────────┐           │
│  │    Jaeger    │ │ Alertmanager │──▶ Slack │
│  └──────────────┘ └──────────────┘           │
└──────────────────────────────────────────────┘
```

> 🚧 일부 컴포넌트는 Phase 4~5에서 도입 예정 (위 Status 참조).

---

## 🛠 Tech Stack

### Application
- **Spring Boot** 3.5.13, **Java** 21 LTS (Eclipse Temurin)
- **DB**: H2 in-memory (local) / PostgreSQL 15 (운영)
- **HTTP**: RestClient (RestTemplate 미사용)
- **Logging**: logstash-encoder 8.0 (JSON 구조화)
- **에러 응답**: RFC 7807 ProblemDetail 표준

### Infrastructure & Cloud
- **AWS**: EKS, VPC, ECR, IAM, S3 (state)
- **IaC**: Terraform 1.14 + AWS Provider ~> 6.0
- **EKS 모듈**: `terraform-aws-modules/eks/aws ~> 21.0`

### Orchestration & Deployment
- **Kubernetes**: 1.33 (AL2023 노드, t3.large × 2)
- **권한 모델**: EKS Pod Identity (IRSA 미사용 — 2026 표준)
- **Packaging**: Kustomize (자체 앱) + Helm (3rd party 차트)
- **CI**: GitHub Actions + OIDC 페더레이션 (정적 자격증명 0개)
- **CD / GitOps**: ArgoCD (Phase 6 도입 예정)

### Observability (3-Pillar) — Phase 4 도입 예정
- **Metrics**: Prometheus + Grafana (kube-prometheus-stack)
- **Logs**: Elasticsearch + Fluent Bit + Kibana
- **Traces**: Jaeger + OpenTelemetry

### Network & Security
- **Ingress**: ALB Ingress Controller (Phase 3.4.6 도입 예정)
- **Service Mesh**: Istio mTLS STRICT, VirtualService 기반 Canary (Phase 5)
- **Image Scan**: Trivy (CI 빌드 시점) + ECR scan_on_push (다층 방어)
- **Image 정책**: IMMUTABLE 태그, untagged 1일/tagged 10개 lifecycle
- **Supply Chain**: Docker BuildKit attestation + SBOM 자동 생성 (SLSA 권장)

---

## 📊 Key Achievements

운영 수치를 측정 가능한 형태로 박제. Phase별로 채워나감.

### 측정 완료 (현재)
- ✅ **Docker 이미지 사이즈**: 130MB (순진한 빌드 266MB 대비 **51% 감소**)
- ✅ **Trivy 보안 스캔**: 3개 이미지 모두 CRITICAL **0개** (HIGH 2개는 영향도 평가 후 baseline 등록)
- ✅ **Terraform 자원 수**: 64개 (VPC + EKS + ECR + 5개 ECR 자원)
- ✅ **인프라 apply 시간**: ~13분 (EKS Control Plane 생성 ~9분이 가장 오래)
- ✅ **K8s Pod 부팅**: ~50초 (Spring Boot 3.5.13 + Alpine JRE)
- ✅ **AWS 운영비**: 매일 destroy 사이클 운영 → 월 $30~40 (Budgets $80 알림)

### 측정 예정 (Phase 4~5)
- 🎯 Fluentd → Fluent Bit 전환: DaemonSet 메모리 절감률
- 🎯 CI 파이프라인 실행 시간 (Layered JAR + 캐싱)
- 🎯 배포 리드 타임 (commit → prod)
- 🎯 SLA 99.9% 산출 기간 동안 가용성

---

## 📝 Architecture Decision Records

운영 환경에서 내린 기술 선택을 ADR로 박제. 총 18개 (앱 10 + 인프라 8).

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

---

## 🔧 Troubleshooting Notes

구축 중 마주친 실제 이슈와 해결 과정 기록입니다.

> 🚧 작성 예정 (Phase 3~6에서 발생하는 사례를 정리)
>
> 현재까지 박제된 함정 28개는 [PROJECT_CONTEXT.md](./PROJECT_CONTEXT.md)의 `함정/주의사항`
> 섹션에 인덱스되어 있으며, 그 중 면접 어필 가치가 큰 사례를 별도 Runbook으로 분리 예정.

---

## 🎬 Screenshots

> 🚧 Phase 4 Observability 도입 후 채울 예정.

<!--
<details>
<summary>Grafana 운영 대시보드</summary>

![Grafana](./docs/screenshots/grafana-ops.png)

</details>

<details>
<summary>ArgoCD 배포 화면</summary>

![ArgoCD](./docs/screenshots/argocd-sync.png)

</details>

<details>
<summary>Kibana 로그 추적</summary>

![Kibana](./docs/screenshots/kibana-trace.png)

</details>
-->

---

## 📬 Contact

- 📧 Email: melanie0617@gmail.com
- 💼 LinkedIn: [linkedin.com/in/melanie-woo](https://www.linkedin.com/in/melanie-woo-32a268229/)
- 📝 Blog: [Notion Blog](https://0617.notion.site/Melanie-s-Devlog-3252455b985d4c0584eb9a94e54a6416?source=copy_link)
