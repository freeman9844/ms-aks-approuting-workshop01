# AKS Application Routing Gateway API 핸즈온 워크숍

Azure Kubernetes Service(AKS)의 **Application Routing 애드온**과 **Gateway API**를 사용해 httpbin 샘플 앱을 HTTP로 노출하고, Azure DNS 및 Key Vault TLS까지 적용하는 실습형 워크숍입니다. 전 과정을 Azure Cloud Shell(bash)에서 `az` CLI와 `kubectl`만으로 진행하며, 코어 실습 시간은 약 **1시간 10분–1시간 30분**입니다.

---

## 아키텍처

```mermaid
flowchart LR
  user([사용자]) -->|"HTTP → HTTPS"| pip["정적 공인 IP"] --> gw["Gateway (approuting-istio)<br/>httpbin-gateway"]
  gw --> route[HTTPRoute] --> svc[httpbin Service]
  subgraph aks ["AKS Standard (Korea Central)"]
    gw; route; svc
    edns["managed external-dns<br/>(옵션)"]
  end
  edns -.->|"A 레코드 자동 발행 (옵션)"| zone[(Azure DNS zone)]
  pip -.->|A 레코드 수동 등록| zone
  gw -.->|인증서 동기화<br/>SecretProviderClass| kv[(Key Vault<br/>자체 서명 인증서)]
  uami[UAMI + 페더레이션<br/>자격 증명] -.-> edns
  uami -.-> gw
```

---

## 학습 목표

1. Managed Gateway API CRD·Application Routing Gateway API·Workload Identity·Key Vault CSI 기능을 켠 AKS 클러스터를 `az` CLI로 생성한다.
2. Gateway와 HTTPRoute로 앱을 HTTP로 노출하고, 자동 생성되는 리소스(GatewayClass·LoadBalancer 서비스 등)의 동작을 이해하며, `spec.addresses`로 Gateway 외부 IP를 정적 공인 IP로 고정한다.
3. HTTPRoute의 `hostname`·`path` 매칭 동작을 직접 관찰한다.
4. Azure DNS zone·Key Vault 인증서·UAMI·페더레이션 자격 증명(FIC)으로 operator 통합 인프라를 구성한다.
5. Gateway annotation 기반 TLS 종료와 정적 IP 기반 A 레코드 등록을 확인한다. (옵션: ClusterExternalDNS를 통한 A 레코드 자동 발행)
6. (옵션) 자동 생성되는 Gateway 인프라(Service·Deployment·HPA·PDB)를 ConfigMap으로 재정의하고, Annotation으로 내부 Load Balancer 전환을 수행한다.
7. (옵션) ingress-nginx와 Gateway API 데이터 플레인을 병렬 구성하고, Azure Front Door 가중치 라우팅으로 카나리 마이그레이션을 수행한다.
8. 실습에 사용한 모든 Azure 리소스를 완전히 정리한다.

---

## 사전 요구사항

| 항목 | 최소 요건 |
|------|-----------|
| Azure 구독 | Owner 또는 (RBAC Administrator + Managed Identity Contributor) 역할 보유 |
| 터미널 | Azure Cloud Shell(bash) — 별도 설치 불필요 |
| az CLI | ≥ 2.86.0 (Cloud Shell 신규 세션에서 확인) |
| Kubernetes 기초 | 파드·서비스 개념 이해 |

---

## 모듈 목차

| 번호 | 제목 | 설명 |
|------|------|------|
| 01 | [사전 준비](docs/01-prerequisites.md) | Cloud Shell 접속, 구독 선택, az CLI 버전 확인 |
| 02 | [환경 준비](docs/02-environment-setup.md) | 리소스 그룹·AKS 클러스터 생성, 자격 증명 취득 |
| 03 | [Gateway·HTTPRoute로 HTTP 노출](docs/03-gateway-httproute.md) | Gateway·HTTPRoute 매니페스트 적용, LB IP 확인·동작 검증, 정적 공인 IP 고정 |
| 04 | [DNS·TLS 인프라 준비](docs/04-dns-tls-infra.md) | DNS zone·Key Vault·UAMI·FIC 생성 및 RBAC 구성 |
| 05 | [TLS Gateway와 DNS A 레코드](docs/05-tls-gateway-externaldns.md) | TLS Gateway 적용, 정적 IP로 A 레코드 등록 (옵션: ClusterExternalDNS 자동 발행) |
| 06 | [Gateway 인프라 커스터마이징 (옵션)](docs/06-gateway-customizations.md) | ConfigMap으로 HPA·Deployment 재정의, Annotation으로 내부 LB 전환 |
| 07 | [AFD 카나리 마이그레이션 (옵션)](docs/07-afd-canary-migration.md) | ingress-nginx ∥ Gateway API 병렬 구성, Azure Front Door 가중치 카나리로 무중단 이관 |
| 08 | [정리](docs/08-cleanup.md) | 전체 Azure 리소스 삭제 |

---

## 시간표

| 모듈 | 제목 | 예상 시간 | 주 소요 요인 |
|------|------|-----------|-------------|
| 01 | 사전 준비 | 5–10분 | Cloud Shell 초기 로딩 |
| 02 | 환경 준비 | 15–20분 (AKS 생성 대기 5–10분 포함) | AKS 클러스터 프로비저닝 |
| 03 | Gateway·HTTPRoute로 HTTP 노출 | 15–20분 (LB IP 할당·정적 IP 구성 대기 포함) | Azure LB 프로비저닝 |
| 04 | DNS·TLS 인프라 준비 | 15–20분 (RBAC 전파 대기 포함) | RBAC 전파 지연 |
| 05 | TLS Gateway와 DNS A 레코드 | 10–15분 (인증서 동기화 대기 포함, ExternalDNS 옵션 +10분) | Key Vault 동기화 |
| 06 | Gateway 인프라 커스터마이징 (옵션) | 10–15분 | HPA 반영·내부 LB 재구성 대기 |
| 07 | AFD 카나리 마이그레이션 (옵션) | 50–80분 | AFD 라우트·가중치 변경 전파 대기 (회당 5–20분) |
| 08 | 정리 | 5–10분 (RG 삭제 완료 대기 포함) | AKS 노드 RG 연쇄 삭제 |
| **합계** | | **≈ 1시간 10분–1시간 30분 (06 옵션 +10–15분, 07 옵션 +50–80분)** | |

---

## 비용 개요

실습 과정에서 생성되는 주요 과금 리소스는 다음과 같습니다.

| 리소스 | 사양 | 비고 |
|--------|------|------|
| AKS 노드 | Standard_D2s_v5 × 2 | 실습 1.5시간 기준 소액 |
| Azure Standard LB | — | Gateway 노출 시 자동 생성 |
| Azure DNS zone | — | 공개 zone 기준 |
| Azure Key Vault | — | 자체 서명 인증서 1건 |
| Azure Front Door Standard (옵션 07) | 기본요금 약 $35/월의 일할 + 요청·전송량 | 실습 1–1.5시간 기준 소액, 07 수행 시에만 생성 |

실습 종료 후 반드시 **08 — 정리** 모듈을 실행해 모든 리소스를 삭제하세요.

---

## 태깅 범례

모든 리소스는 워크샵 전용 리소스 그룹에 생성되며, RG에 다음 태그를 부여해 용도를 식별합니다.

| 태그 | 값 | 목적 |
|------|-----|------|
| `workload` | `aks-approuting-workshop` | 워크샵 리소스 식별 |
| `environment` | `workshop` | 임시 실습 환경임을 표시 (정리 대상) |

AKS 노드 리소스 그룹(`MC_...`) 내부 리소스는 AKS가 자동 관리하므로 별도 태깅하지 않습니다.

---

## 참고 자료

### AKS Application Routing·Gateway API (Microsoft Learn)

- [AKS Application Routing — Gateway API 사용](https://learn.microsoft.com/en-us/azure/aks/app-routing-gateway-api) — 이 워크샵 03·05 모듈의 원본 문서
- [AKS Application Routing — DNS·TLS 구성](https://learn.microsoft.com/en-us/azure/aks/app-routing-gateway-api-dns-tls) — 04·05 모듈의 원본 문서
- [AKS Application Routing 애드온 개요](https://learn.microsoft.com/en-us/azure/aks/app-routing)
- [AKS Managed Gateway API](https://learn.microsoft.com/en-us/azure/aks/managed-gateway-api)
- [Istio Gateway API — ConfigMap·Annotation customizations](https://learn.microsoft.com/en-us/azure/aks/istio-gateway-api#configmap-customizations) — 06 모듈의 원본 문서
- [AKS에서 정적 IP로 Load Balancer 사용](https://learn.microsoft.com/en-us/azure/aks/static-ip) — 03 모듈 8절 배경

### 마이그레이션·Azure Front Door (Microsoft Learn) — 07 모듈

- [ingress-nginx에서 Gateway API로 마이그레이션](https://learn.microsoft.com/en-us/azure/aks/app-routing-nginx-to-gateway-api-migration) — 07 모듈의 원본 문서
- [Azure Front Door를 이용한 Blue/Green 배포](https://learn.microsoft.com/en-us/azure/frontdoor/blue-green-deployment)
- [Azure Front Door 트래픽 라우팅 방법 (가중치 라우팅)](https://learn.microsoft.com/en-us/azure/frontdoor/routing-methods#weighted-traffic-routing-method)

### 자격 증명·인증서 (Microsoft Learn)

- [AKS Workload Identity 개요](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview) — 04 모듈 UAMI·FIC의 동작 원리
- [Workload Identity Federation (Microsoft Entra)](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation)
- [AKS Key Vault Secrets Store CSI Driver](https://learn.microsoft.com/en-us/azure/aks/csi-secrets-store-driver) — 05 모듈 인증서 동기화 메커니즘
- [Key Vault 인증서 개요](https://learn.microsoft.com/en-us/azure/key-vault/certificates/about-certificates)
- [Key Vault soft-delete 개요](https://learn.microsoft.com/en-us/azure/key-vault/general/soft-delete-overview) — 08 모듈 purge가 필요한 이유

### DNS·네트워킹 (Microsoft Learn)

- [Azure DNS zone과 레코드 개요](https://learn.microsoft.com/en-us/azure/dns/dns-zones-records)
- [Azure 공인 IP 주소 (SKU·할당 방식)](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/public-ip-addresses)
- [az aks CLI 레퍼런스](https://learn.microsoft.com/en-us/cli/azure/aks)

### 오픈소스 공식 문서

- [Gateway API 공식 문서](https://gateway-api.sigs.k8s.io/)
- [ExternalDNS 공식 문서](https://kubernetes-sigs.github.io/external-dns/latest/) — 05 모듈 옵션의 upstream 프로젝트
- [Istio httpbin 샘플](https://github.com/istio/istio/tree/master/samples/httpbin) — 03 모듈에서 배포하는 샘플 앱
