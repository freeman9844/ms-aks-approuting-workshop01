# AKS Application Routing Gateway API 워크샵 — 디자인 스펙

- 날짜: 2026-07-25
- 상태: 승인됨 (사용자 리뷰 완료 전)
- 대상 저장소: `ms-aks-approuting-workshop01` (신규)

## 1. 목적과 범위

AKS **application routing add-on의 Gateway API 구현**(Istio 기반, `gatewayClassName: approuting-istio`)을
az CLI + Azure Cloud Shell(bash)로 체험하는 한국어 핸즈온 워크샵.

- 근거 문서:
  - https://learn.microsoft.com/en-us/azure/aks/app-routing-gateway-api
  - https://learn.microsoft.com/en-us/azure/aks/app-routing-gateway-api-dns-tls
  - https://learn.microsoft.com/en-us/azure/aks/managed-gateway-api
- 분량: 코어 6모듈, 약 1시간 10분–1시간 30분 (선택 모듈 없음)
- 대상: 인프라/플랫폼 엔지니어. Owner(또는 RBAC Administrator + Managed Identity Contributor) 구독 필요
  — 모듈 04에서 역할 할당·페더레이션 자격 증명을 생성하기 때문.
- 지역: Korea Central. 도구: Cloud Shell의 `az`(≥ 2.86.0) · `kubectl` · `curl` · `dig`.

### 범위 제외 (YAGNI)

- Terraform / WAF·CAF 섹션 없음 (ACA/App Service 사이블링 계열의 순수 az CLI 스타일)
- 트래픽 분할, 오토스케일, 모니터링, namespace-scoped ExternalDNS, 실제 도메인 위임 없음
- 샘플 앱은 Learn 문서 그대로 **httpbin** (Istio 릴리스 매니페스트) — 별도 앱 소스 없음

## 2. 핵심 설계 결정

| 결정 | 선택 | 근거 |
|---|---|---|
| 저장소 | 신규 폴더 `~/general/ms-aks-approuting-workshop01` | 기존 Terraform 워크샵과 성격이 다름 |
| DNS·TLS 검증 방식 | 위임 없는 Azure DNS zone + Key Vault 자체 서명 인증서 | 수강생 도메인 보유 전제 제거 — 누구나 수행 가능 |
| 도메인 | 가상 도메인(예: `ws<suffix>.example-workshop.com`), 실제 등록 불필요 | 위임하지 않으므로 충돌 무해; 검증은 zone 네임서버 직접 질의 |
| 검증 명령 | `dig @<zone NS>` (A 레코드) + `curl -k --resolve` (HTTPS) | 공인 리졸버를 거치지 않고 end-to-end 확인 |
| 클러스터 | AKS Standard, **생성 시점에** 필요한 기능 전부 활성화 | `az aks update` GET→PUT 왕복의 설정 유실 리스크 회피 |
| 샘플 앱 | httpbin (Learn 문서 그대로) | 가장 단순, 문서와 1:1 대응 |
| ExternalDNS 종류 | `ClusterExternalDNS` (클러스터 스코프)만 | 코어 분량 유지; namespace-scoped는 참고 링크로만 언급 |
| 모듈 구조 | DNS·TLS를 "04 인프라 준비"와 "05 구성·검증"으로 분리 | 모듈당 10–20분 리듬 유지; 04의 Azure 리소스 생성 대기를 ⏳ 읽을거리로 흡수 |

## 3. 저장소 구조

```
README.md            # 사이블링 캐노니컬 스켈레톤(아래 참조)
docs/
  01-prerequisites.md
  02-environment-setup.md
  03-gateway-httproute.md
  04-dns-tls-infra.md
  05-tls-gateway-externaldns.md
  06-cleanup.md
manifests/
  gateway-http.yaml          # 모듈 03: Gateway + HTTPRoute (HTTP)
  gateway-tls.yaml           # 모듈 05: HTTPS listener + KV 인증서 annotation
  cluster-external-dns.yaml  # 모듈 05: ClusterExternalDNS (플레이스홀더 치환 후 적용)
.gitignore           # session*.md 등 (스펙·플랜은 docs/superpowers/에 커밋)
LICENSE              # MIT (사이블링과 동일)
```

README 순서(캐노니컬): 아키텍처(mermaid) → 학습 목표(번호 목록) → 사전 요구사항(표) →
모듈 목차(코어만, 표) → 시간표 → 비용 개요 → 트러블슈팅 색인 → 참고 자료.

## 4. 문서 컨벤션 (ACA/App Service 사이블링과 동일)

- 한국어, Cloud Shell(bash) 전용, 복붙 가능해야 함.
- 각 문서 상단 한 줄 범례: `🟢 실행 = 직접 입력·수행 · 👁️ 예시 = 눈으로만(개념/발췌) · 📋 예상 출력 = 비교용(입력 불필요)`.
  모든 코드 펜스 바로 위에 라벨 한 줄.
- 각 모듈 시작에 선택적 "0단계 — 변수 재설정" 블록(새 터미널 대비 `SUFFIX`·환경변수 복원).
- 각 모듈 끝에 트러블슈팅 3열 표(증상 · 원인 · 해결 방법)와 이전/다음 모듈 네비게이션 링크.
- 시간 추정은 실측 범위(라운드 넘버 금지) + "주 소요 요인" 열.
- 숫자 범위는 en-dash `–` 사용(`~` 금지).
- 긴 대기(⏳)에는 "기다리는 동안 읽기" 개념 설명 배치.
- 리소스 명명: `SUFFIX`(랜덤 5자리) 기반 — 예: `rg-approuting-ws-<suffix>`, `aks-approuting-<suffix>`,
  `kv-approuting-<suffix>`(KV는 전역 고유·24자 제한 주의), zone `ws<suffix>.example-workshop.com`.
- 재실행 가능(멱등) 명령 우선: `kubectl create ... --dry-run=client -o yaml | kubectl apply -f -` 패턴 등.

## 5. 모듈별 상세

### 01 — 사전 준비 (~10분)

- Cloud Shell(bash) 접속, 구독 선택·확인.
- `az version` 확인 — **2.86.0 이상 필수** (`--enable-gateway-api`, `--enable-app-routing-istio` 플래그 요구).
  Cloud Shell 버전이 낮은 경우의 대응(세션 재시작, 반영 지연 안내)을 트러블슈팅에 수록.
- `SUFFIX=$RANDOM` 계열 생성, 환경변수 세트 정의(RG, CLUSTER, LOCATION=koreacentral, ZONE_NAME, KV_NAME, UAMI_NAME).

### 02 — 환경 준비 (15–20분, AKS 생성 대기 5–10분)

- RG 생성.
- 클러스터 생성 **한 번의 `az aks create`로** 모든 전제 조건 활성화:

  ```
  az aks create \
    --resource-group $RESOURCE_GROUP --name $CLUSTER --location koreacentral \
    --node-count 2 \
    --enable-gateway-api \
    --enable-app-routing-istio \
    --enable-oidc-issuer \
    --enable-workload-identity \
    --enable-addons azure-keyvault-secrets-provider \
    --generate-ssh-keys
  ```

  (노드 크기·정확한 플래그 조합은 리허설에서 확정. `--enable-app-routing-istio`가 az stable에 없으면
  preview extension 설치 단계를 01에 추가.)
- ⏳ 대기 중 읽을거리: Gateway API란(Ingress API의 한계, Ingress NGINX 2026-03 은퇴 배경),
  application routing Gateway API vs Istio 메시 애드온 비교, Managed Gateway API CRD 개념.
- `az aks get-credentials` → 검증:
  - `kubectl get crds | grep gateway.networking.k8s.io`
  - `kubectl get pods -n aks-istio-system` (istiod)
  - `kubectl get gatewayclass` (`approuting-istio`)

### 03 — Gateway·HTTPRoute로 HTTP 노출 (10–15분, LB IP 대기)

- httpbin 배포(Learn 문서 그대로, `ISTIO_RELEASE` 고정):
  `kubectl apply -f https://raw.githubusercontent.com/istio/istio/$ISTIO_RELEASE/samples/httpbin/httpbin.yaml`
- `manifests/gateway-http.yaml` 적용: `gatewayClassName: approuting-istio` Gateway + HTTPRoute.
- Gateway의 LB External IP 대기·확인 (자동 생성 서비스 명명 규칙 `<gateway>-approuting-istio`를 본문에서 설명).
- `curl http://$GATEWAY_IP/headers` 등으로 HTTP 200 + 응답 관찰. HTTPRoute 매칭(경로/헤더) 가벼운 실험 1개.

### 04 — DNS·TLS 인프라 준비 (15–20분, RBAC 전파 대기)

Learn DNS·TLS 문서의 "Create the Azure infrastructure" 절에 대응:

1. Azure DNS zone 생성 (`az network dns zone create`, 가상 도메인) — 위임하지 않음을 명시하고 이유 설명.
2. Key Vault 생성(RBAC 모드) + 자체 서명 인증서 발급 (`az keyvault certificate create`, 기본 정책에 CN=도메인).
3. UAMI 생성 + 역할 할당 2건: zone에 **DNS Zone Contributor**, KV에 **Key Vault Secrets User**.
   (실습자 본인에게 KV Certificates Officer 등 발급 권한 부여가 필요하면 리허설에서 확인 후 추가.)
4. 네임스페이스·ServiceAccount 생성, UAMI에 페더레이션 자격 증명 2건(cert 동기화용 / external-dns용 —
   정확한 구성은 Learn 문서 절차를 따름).
- ⏳ RBAC 전파(최대 ~5분) 대기 중 읽을거리: workload identity·페더레이션 자격 증명 개념,
  Application Routing operator가 SecretProviderClass→Secret→certificateRefs를 reconcile하는 흐름.

### 05 — TLS Gateway + ClusterExternalDNS (15–20분, reconcile·레코드 대기)

1. Gateway에 HTTPS listener(443) + Key Vault 인증서 URI annotation 적용 (`manifests/gateway-tls.yaml`).
2. 관찰: SecretProviderClass·TLS Secret이 자동 생성되고 listener `certificateRefs`가 채워지는 것 확인.
3. `ClusterExternalDNS` 적용 (`manifests/cluster-external-dns.yaml`, zone resource ID·identity 플레이스홀더 치환 —
   `sed` 사용 시 구분자 `#` 등 치환 변수와 충돌하지 않는 문자 사용).
4. A 레코드 생성 확인: `az network dns record-set a list` + zone 네임서버 직접 질의
   `dig @<zone NS 1번> httpbin.$ZONE_NAME`.
5. HTTPS end-to-end 검증: `curl -k --resolve httpbin.$ZONE_NAME:443:$GATEWAY_IP https://httpbin.$ZONE_NAME/headers`.
   **교육 포인트**: `-k`가 필요한 이유(자체 서명), 실제 운영에서는 위임된 도메인 + 공인 CA 인증서 사용.

### 06 — 정리 (~5분)

- `ClusterExternalDNS`가 남긴 A 레코드는 RG 삭제로 함께 제거됨(모두 동일 RG).
- `az group delete --no-wait` + 삭제 확인 방법. KV soft-delete/purge 안내(이름 재사용 시 purge 필요).
- 과금 종료 확인 안내.

## 6. 리스크와 완화

| 리스크 | 완화 |
|---|---|
| Cloud Shell az < 2.86.0 | 01에서 버전 게이트; 트러블슈팅에 대응 절차 |
| `--enable-app-routing-istio` 등 플래그가 preview extension 요구 | 리허설에서 확인; 필요 시 01에 extension 설치 단계 추가 |
| RBAC/페더레이션 자격 증명 전파 지연 | ⏳ 읽을거리로 흡수 + 트러블슈팅에 재시도 안내 |
| KV 이름 전역 고유·24자 제한 | SUFFIX 조합 규칙으로 길이 보장 |
| `az aks update` 왕복 설정 유실 | 생성 시점 일괄 활성화로 원천 회피 |
| 예상 출력이 실제와 다름 | **전 명령 실 계정 리허설 후 📋 블록 작성** — 추정 출력 금지 |

## 7. 검증 계획 (콘텐츠 완성 후, 릴리스 전)

1. 실제 구독에서 01→06 전체 복붙 리허설 1회 — 예상 출력 캡처, 시간 실측 → 시간표 보정.
2. 정적 검사: 짝수 코드 펜스(`grep -c '^\`\`\`'`), 외부 링크 상태 코드(curl), README 목차·시간표·파일명 일치.
3. 라벨 검사: 모든 코드 펜스 위에 🟢/👁️/📋 라벨 존재.

## 8. 시간표 (초안 — 리허설 후 실측으로 보정)

| 모듈 | 제목 | 예상 시간 | 주 소요 요인 |
|---|---|---|---|
| 01 | 사전 준비 | ~10분 | Cloud Shell 기동·버전 확인 |
| 02 | 환경 준비 | 15–20분 | AKS 생성 대기 5–10분 |
| 03 | Gateway·HTTPRoute | 10–15분 | LB IP 할당 대기 |
| 04 | DNS·TLS 인프라 | 15–20분 | RBAC 전파 |
| 05 | TLS Gateway + ExternalDNS | 15–20분 | Secret reconcile·A 레코드 대기 |
| 06 | 정리 | ~5분 | RG 삭제(비동기) |
| | **전체** | **≈ 1시간 10분–1시간 30분** | |

## 9. 비용 개요

Standard AKS(시스템 노드 2대, D2 계열) + Standard LB + 공용 DNS zone + Key Vault.
실습 1.5시간 기준 소액(수천 원 미만 수준 — 리허설 후 실측 기입). 06 정리 필수 강조.
