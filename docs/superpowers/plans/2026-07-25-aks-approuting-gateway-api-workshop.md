# AKS Application Routing Gateway API 워크샵 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** AKS application routing add-on의 Gateway API 구현(HTTP → DNS·TLS·HTTPS)을 az CLI + Cloud Shell로 체험하는 한국어 핸즈온 워크샵 콘텐츠(6모듈 + README + 매니페스트) 작성.

**Architecture:** 문서 저장소(코드 아님). `docs/01–06` 모듈 + `manifests/` YAML + README. 스펙은 `docs/superpowers/specs/2026-07-25-aks-approuting-gateway-api-workshop-design.md`. 검증은 정적 검사(코드 펜스 짝수, 라벨 존재, 링크 상태, 파일명·목차 일치)이며, Azure 실 리허설은 계획 범위 밖(작성 후 별도 수행)이다.

**Tech Stack:** Markdown(한국어), Kubernetes YAML, az CLI ≥ 2.86.0, bash.

## Global Constraints

스펙에서 그대로 가져온 프로젝트 전역 규칙 — 모든 태스크에 암묵적으로 적용:

- 언어: 한국어. 실행 환경: Azure Cloud Shell(bash). 지역: `koreacentral`. 모든 명령은 복붙 가능해야 함.
- 모든 코드 펜스 **바로 위 한 줄**에 라벨: `🟢 **실행**` / `👁️ **예시**` / `📋 **예상 출력**`. 각 문서 상단에 범례 한 줄: `> 🟢 실행 = 직접 입력·수행 · 👁️ 예시 = 눈으로만(개념/발췌) · 📋 예상 출력 = 비교용(입력 불필요)`.
- 📋 예상 출력은 리허설 전이므로 Learn 문서의 output 블록을 기반으로 작성하고, 문서 맨 끝(네비게이션 링크 아래 아님, 파일 끝)에 HTML 주석 `<!-- TODO(rehearsal): 예상 출력 실측 검증 -->`을 넣는다. 이 주석은 리허설 단계에서 제거한다. 이것은 유일하게 허용되는 TODO다.
- 숫자 범위는 en-dash `–` 사용, `~단독범위` 표기 외 물결(`~`) 금지 (GFM 취소선 회피). 단, "약"의 의미로 쓰는 `~5분`은 허용.
- 각 모듈(02–06) 시작에 "0단계 — 변수 재설정" 접기 블록(`<details>`), 끝에 트러블슈팅 3열 표(증상 · 원인 · 해결 방법)와 이전/다음 네비게이션 링크.
- 모듈 01에서 정의하는 표준 환경변수 세트(모든 문서에서 동일 이름 사용):
  `SUFFIX`, `RESOURCE_GROUP=rg-approuting-ws-$SUFFIX`, `CLUSTER=aks-approuting-$SUFFIX`, `LOCATION=koreacentral`, `ZONE_NAME=ws$SUFFIX.approuting-workshop.example`, `KV_NAME=kv-apr-$SUFFIX`, `UAMI_NAME=id-approuting-$SUFFIX`, `SA_NAME=approuting-ws-sa`, `APP_NAMESPACE=workshop`, `CERT_NAME=approuting-ws-cert`.
- Git 커밋: conventional-commit 접두사 + 한국어 제목 (예: `docs(03): ...`), Co-authored-by 트레일러 포함.
- 시간 추정은 범위로 표기(라운드 넘버 총합 금지), README 시간표와 각 모듈 도입부의 추정치는 항상 동일해야 함.
- 근거 Learn 문서 3편(모든 명령의 출처 — 임의 변형 금지, 워크샵 변수명 치환만 허용):
  - https://learn.microsoft.com/en-us/azure/aks/app-routing-gateway-api
  - https://learn.microsoft.com/en-us/azure/aks/app-routing-gateway-api-dns-tls
  - https://learn.microsoft.com/en-us/azure/aks/managed-gateway-api

**스펙 대비 단순화(승인된 코어 범위):** Learn DNS·TLS 문서는 `app-a`/`app-b` 두 네임스페이스 데모지만, 본 워크샵은 **단일 네임스페이스 `workshop` + 단일 Gateway** 로 단순화한다(1.5시간 코어 목표). FIC도 1건. 호스트명은 `httpbin.$ZONE_NAME`.

---

### Task 1: 저장소 뼈대 + 모듈 01 (사전 준비)

**Files:**
- Create: `.gitignore`
- Create: `LICENSE`
- Create: `docs/01-prerequisites.md`

**Interfaces:**
- Produces: 표준 환경변수 세트와 `~/.approuting-ws-env` 저장 패턴(이후 모든 모듈의 "0단계"가 `source ~/.approuting-ws-env`로 복원). 파일명 `docs/01-prerequisites.md`.

- [ ] **Step 1: .gitignore와 LICENSE 작성**

`.gitignore`:

```
session*.md
cert-policy.json
*.pem
.DS_Store
```

`LICENSE`: MIT 전문, `Copyright (c) 2026 Jungwoon Lee`.

- [ ] **Step 2: docs/01-prerequisites.md 작성**

구성(이 순서대로):

1. H1: `# 01 — 사전 준비` + 라벨 범례 한 줄 + 예상 시간 `~10분`.
2. **Cloud Shell 접속**: portal.azure.com → Cloud Shell(Bash) 안내, 구독 확인:

   🟢 실행

   ```bash
   az account show --query '{name:name, id:id}' -o table
   # 구독이 여러 개면: az account set --subscription "<구독 이름 또는 ID>"
   ```

3. **az CLI 버전 게이트** (2.86.0 이상 필수 — `--enable-gateway-api`·`--enable-app-routing-istio` 플래그 요구):

   🟢 실행

   ```bash
   az version --query '"azure-cli"' -o tsv
   ```

   📋 예상 출력

   ```
   2.86.0
   ```

   버전이 낮을 때의 대응(Cloud Shell은 자동 업데이트되나 반영 지연 가능 → 세션 재시작 후 재확인)을 본문에 기술.
4. **환경변수 정의 및 저장** — Global Constraints의 표준 변수 세트를 그대로 사용:

   🟢 실행

   ```bash
   export SUFFIX=$(printf '%05d' $((RANDOM % 100000)))
   export RESOURCE_GROUP=rg-approuting-ws-$SUFFIX
   export CLUSTER=aks-approuting-$SUFFIX
   export LOCATION=koreacentral
   export ZONE_NAME=ws$SUFFIX.approuting-workshop.example
   export KV_NAME=kv-apr-$SUFFIX
   export UAMI_NAME=id-approuting-$SUFFIX
   export SA_NAME=approuting-ws-sa
   export APP_NAMESPACE=workshop
   export CERT_NAME=approuting-ws-cert

   cat > ~/.approuting-ws-env <<EOF
   export SUFFIX=$SUFFIX
   export RESOURCE_GROUP=$RESOURCE_GROUP
   export CLUSTER=$CLUSTER
   export LOCATION=$LOCATION
   export ZONE_NAME=$ZONE_NAME
   export KV_NAME=$KV_NAME
   export UAMI_NAME=$UAMI_NAME
   export SA_NAME=$SA_NAME
   export APP_NAMESPACE=$APP_NAMESPACE
   export CERT_NAME=$CERT_NAME
   EOF
   echo "SUFFIX=$SUFFIX 저장 완료"
   ```

   본문에 설명: `KV_NAME=kv-apr-$SUFFIX`는 최대 12자로 Key Vault 24자·전역 고유 제한을 만족. `ZONE_NAME`은 **위임하지 않는 가상 도메인**(실제 등록 불필요 — 검증은 zone 네임서버 직접 질의)임을 명시.
5. 트러블슈팅 표(3열): az 버전이 낮음 / Cloud Shell 스토리지 미구성 / 구독 권한 부족(Owner 또는 RBAC Administrator + Managed Identity Contributor 필요 — 04에서 역할 할당·FIC 생성 때문) 3행.
6. 네비게이션: `다음: [02 — 환경 준비](02-environment-setup.md)` (01은 이전 링크 없음, README로 링크).

- [ ] **Step 3: 정적 검사**

Run: `cd ~/general/ms-aks-approuting-workshop01 && test $(( $(grep -c '^```' docs/01-prerequisites.md) % 2 )) -eq 0 && echo FENCES-OK`
Expected: `FENCES-OK`

- [ ] **Step 4: Commit**

```bash
git add .gitignore LICENSE docs/01-prerequisites.md
git commit -m "docs(01): 저장소 뼈대 및 사전 준비 모듈 작성

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

---

### Task 2: 모듈 02 (환경 준비 — AKS 클러스터 + 전 기능 활성화)

**Files:**
- Create: `docs/02-environment-setup.md`

**Interfaces:**
- Consumes: Task 1의 환경변수 세트, `source ~/.approuting-ws-env` 패턴.
- Produces: 기능이 모두 켜진 AKS 클러스터(문서 절차상). 파일명 `docs/02-environment-setup.md`. "0단계 — 변수 재설정" `<details>` 블록 패턴(이후 태스크가 동일 형식 복제).

- [ ] **Step 1: docs/02-environment-setup.md 작성**

구성:

1. H1 `# 02 — 환경 준비` + 범례 + 예상 시간 `15–20분 (AKS 생성 대기 5–10분 포함)`.
2. **0단계 — 변수 재설정** (이 블록 형식을 03–06에서도 동일하게 사용):

   ```markdown
   <details>
   <summary>🔄 0단계 — 변수 재설정 (새 터미널/세션에서 시작하는 경우)</summary>

   🟢 **실행**

   ```bash
   source ~/.approuting-ws-env
   az aks get-credentials --resource-group $RESOURCE_GROUP --name $CLUSTER --overwrite-existing 2>/dev/null || true
   echo "RESOURCE_GROUP=$RESOURCE_GROUP"
   ```

   </details>
   ```

   (02의 0단계는 `get-credentials` 줄 없이 `source`와 `echo`만 — 클러스터가 아직 없음.)
3. RG 생성:

   🟢 실행

   ```bash
   az group create --name $RESOURCE_GROUP --location $LOCATION -o table
   ```

4. **클러스터 생성 — 한 번의 `az aks create`로 전 기능 활성화** (왜 한 번에 켜는지 본문 설명 — `az aks update`는 GET→PUT 왕복으로 이전 설정을 유실할 수 있음):

   🟢 실행

   ```bash
   az aks create \
     --resource-group $RESOURCE_GROUP \
     --name $CLUSTER \
     --location $LOCATION \
     --node-count 2 \
     --node-vm-size Standard_D2s_v5 \
     --enable-gateway-api \
     --enable-app-routing-istio \
     --enable-oidc-issuer \
     --enable-workload-identity \
     --enable-addons azure-keyvault-secrets-provider \
     --generate-ssh-keys
   ```

   플래그 6개 각각의 역할을 표(플래그 · 역할 · 어느 모듈에서 사용)로 설명.
5. **⏳ 기다리는 동안 읽기** 섹션(5–10분 분량): ① Gateway API란 — Ingress API의 한계(annotation 파편화, 역할 분리 부재)와 Gateway/HTTPRoute 역할 분리 모델, Ingress NGINX 2026년 3월 유지보수 종료 배경, ② application routing Gateway API vs Istio 서비스 메시 애드온 차이(전자는 ingress 전용 관리형, 메시 기능 없음), ③ Managed Gateway API CRD 설치 개념(standard 채널만, K8s 버전별 번들 버전 자동 관리). 👁️ 예시로 mermaid 다이어그램 1개: `사용자 → Gateway(approuting-istio LB) → HTTPRoute → Service(httpbin)`.
6. 자격 증명 및 검증 3종:

   🟢 실행

   ```bash
   az aks get-credentials --resource-group $RESOURCE_GROUP --name $CLUSTER --overwrite-existing
   kubectl get crds | grep gateway.networking.k8s.io
   kubectl get pods -n aks-istio-system
   kubectl get gatewayclass
   ```

   📋 예상 출력 — CRD 5종(gatewayclasses/gateways/grpcroutes/httproutes/referencegrants), `istiod-*` 파드 2개 Running, gatewayclass `approuting-istio` (Learn 문서 output 기반).
7. 트러블슈팅 표: `--enable-app-routing-istio` unrecognized(az 구버전 → 01 버전 게이트 재확인) / 쿼터 부족(Standard_D2s_v5 vCPU) / istiod 파드 Pending 3행.
8. 네비게이션: 이전 01 / 다음 03.

- [ ] **Step 2: 정적 검사**

Run: `cd ~/general/ms-aks-approuting-workshop01 && test $(( $(grep -c '^```' docs/02-environment-setup.md) % 2 )) -eq 0 && grep -q '0단계' docs/02-environment-setup.md && echo OK`
Expected: `OK`

- [ ] **Step 3: Commit**

```bash
git add docs/02-environment-setup.md
git commit -m "docs(02): 환경 준비 모듈 작성 — AKS 생성 시점 전 기능 활성화

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

---

### Task 3: 모듈 03 (Gateway·HTTPRoute HTTP 노출) + manifests/gateway-http.yaml

**Files:**
- Create: `manifests/gateway-http.yaml`
- Create: `docs/03-gateway-httproute.md`

**Interfaces:**
- Consumes: Task 2의 클러스터, 0단계 블록 패턴(get-credentials 포함 버전).
- Produces: `workshop` 네임스페이스 + httpbin + `httpbin-gateway`(HTTP :80, hostname 없음)와 HTTPRoute `httpbin`(hostnames `["httpbin.example.com"]`). Task 5가 이 Gateway를 HTTPS 버전으로 **대체(apply 덮어쓰기)** 한다. 매니페스트 경로 `manifests/gateway-http.yaml`.

- [ ] **Step 1: manifests/gateway-http.yaml 작성**

Learn 문서 예제를 `workshop` 네임스페이스용으로 작성(네임스페이스는 kubectl `-n` 로 지정, 매니페스트에 metadata.namespace 넣지 않음 — 05에서 동일 이름 덮어쓰기 대비 단순화):

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  gatewayClassName: approuting-istio
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: Same
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: httpbin
spec:
  parentRefs:
  - name: httpbin-gateway
  hostnames: ["httpbin.example.com"]
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /get
    backendRefs:
    - name: httpbin
      port: 8000
```

- [ ] **Step 2: docs/03-gateway-httproute.md 작성**

구성:

1. H1 `# 03 — Gateway·HTTPRoute로 HTTP 노출` + 범례 + 예상 시간 `10–15분 (LB IP 할당 대기 포함)`.
2. 0단계 블록(Task 2 형식, get-credentials 포함).
3. 리포지토리 클론(매니페스트 사용을 위해 — 01이 아닌 여기서 처음 필요):

   🟢 실행

   ```bash
   cd ~ && git clone https://github.com/jungwoonlee_microsoft/ms-aks-approuting-workshop01.git 2>/dev/null || (cd ms-aks-approuting-workshop01 && git pull)
   cd ~/ms-aks-approuting-workshop01
   ```

4. 네임스페이스(멱등) + httpbin 배포(Learn 문서 그대로, 릴리스 고정):

   🟢 실행

   ```bash
   kubectl create namespace $APP_NAMESPACE --dry-run=client -o yaml | kubectl apply -f -
   export ISTIO_RELEASE="release-1.27"
   kubectl apply -n $APP_NAMESPACE -f https://raw.githubusercontent.com/istio/istio/$ISTIO_RELEASE/samples/httpbin/httpbin.yaml
   kubectl get pods -n $APP_NAMESPACE
   ```

5. Gateway·HTTPRoute 적용 — 매니페스트 내용을 먼저 👁️ 예시로 보여주고 필드(gatewayClassName, listeners, parentRefs, hostnames, backendRefs) 설명 후:

   🟢 실행

   ```bash
   kubectl apply -n $APP_NAMESPACE -f manifests/gateway-http.yaml
   kubectl wait -n $APP_NAMESPACE --for=condition=programmed gateway httpbin-gateway --timeout=300s
   ```

6. 자동 생성 리소스 관찰 — 본문에서 명명 규칙 `<gateway 이름>-<gatewayclass 이름>` → `httpbin-gateway-approuting-istio` 설명:

   🟢 실행

   ```bash
   kubectl get deployment,service,hpa,pdb -n $APP_NAMESPACE httpbin-gateway-approuting-istio
   ```

   📋 예상 출력: Learn 문서의 deployment 2/2, LoadBalancer EXTERNAL-IP, hpa 2–5, pdb 표를 통합 형태로 수록.
7. HTTP 요청:

   🟢 실행

   ```bash
   export INGRESS_HOST=$(kubectl get gateway httpbin-gateway -n $APP_NAMESPACE -ojsonpath='{.status.addresses[0].value}')
   echo "Gateway IP: $INGRESS_HOST"
   curl -s -I -HHost:httpbin.example.com "http://$INGRESS_HOST/get"
   ```

   📋 예상 출력: `HTTP/1.1 200 OK` 헤더.
8. **가벼운 실험 — HTTPRoute 매칭 관찰**: Host 헤더 없이/틀린 경로로 보내면 404가 됨을 확인(hostnames·path 매칭 동작 체감):

   🟢 실행

   ```bash
   curl -s -o /dev/null -w "Host 불일치: %{http_code}\n" "http://$INGRESS_HOST/get"
   curl -s -o /dev/null -w "경로 불일치: %{http_code}\n" -HHost:httpbin.example.com "http://$INGRESS_HOST/headers"
   ```

   📋 예상 출력: 두 줄 모두 404.
9. 트러블슈팅 표: EXTERNAL-IP `<pending>` 지속 / Gateway programmed 안 됨(gatewayclass 오타) / curl 404(Host 헤더 누락) 3행.
10. 네비게이션: 이전 02 / 다음 04.

- [ ] **Step 3: 정적 검사 (YAML 유효성 + 펜스)**

Run: `cd ~/general/ms-aks-approuting-workshop01 && python3 -c "import yaml,sys; list(yaml.safe_load_all(open('manifests/gateway-http.yaml')))" && test $(( $(grep -c '^```' docs/03-gateway-httproute.md) % 2 )) -eq 0 && echo OK`
Expected: `OK`

- [ ] **Step 4: Commit**

```bash
git add manifests/gateway-http.yaml docs/03-gateway-httproute.md
git commit -m "docs(03): Gateway·HTTPRoute HTTP 노출 모듈 및 매니페스트 작성

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

---

### Task 4: 모듈 04 (DNS·TLS 인프라 준비)

**Files:**
- Create: `docs/04-dns-tls-infra.md`

**Interfaces:**
- Consumes: Task 1 변수 세트(`ZONE_NAME`, `KV_NAME`, `UAMI_NAME`, `SA_NAME`, `CERT_NAME`, `APP_NAMESPACE`), Task 2 클러스터(OIDC issuer).
- Produces: 문서 절차상 산출물 — DNS zone(`$ZONE_ID`), KV + 인증서(`$CERT_URI`, 무버전 URI), UAMI(`$UAMI_CLIENT_ID`), 역할 할당 2건, `workshop` 네임스페이스의 ServiceAccount `$SA_NAME` + FIC 1건. `~/.approuting-ws-env`에 `ZONE_ID`·`CERT_URI`·`UAMI_CLIENT_ID` 추가 저장(Task 5의 0단계가 복원).

- [ ] **Step 1: docs/04-dns-tls-infra.md 작성**

Learn DNS·TLS 문서 "Create the Azure infrastructure" 절을 단일 네임스페이스로 치환. 구성:

1. H1 `# 04 — DNS·TLS 인프라 준비` + 범례 + 예상 시간 `15–20분 (RBAC 전파 대기 포함)`.
2. 0단계 블록.
3. **DNS zone 생성** — 본문: 위임 없는 가상 도메인 전략 설명(공인 리졸버에는 안 보이지만 zone의 권한 네임서버에 직접 질의하면 확인 가능; 실 운영은 등록기관에서 NS 위임):

   🟢 실행

   ```bash
   az network dns zone create --resource-group $RESOURCE_GROUP --name $ZONE_NAME
   export ZONE_ID=$(az network dns zone show --resource-group $RESOURCE_GROUP --name $ZONE_NAME --query id -o tsv)
   ```

4. **Key Vault(RBAC 모드) 생성 + 본인에게 인증서 발급 권한 부여** (Learn 문서 Note의 요구 — Key Vault Certificates Officer):

   🟢 실행

   ```bash
   az keyvault create \
     --name $KV_NAME \
     --resource-group $RESOURCE_GROUP \
     --location $LOCATION \
     --enable-rbac-authorization true

   export MY_OBJECT_ID=$(az ad signed-in-user show --query id -o tsv)
   az role assignment create \
     --assignee-object-id $MY_OBJECT_ID \
     --assignee-principal-type User \
     --role "Key Vault Certificates Officer" \
     --scope $(az keyvault show --name $KV_NAME --query id -o tsv)
   ```

5. **자체 서명 와일드카드 인증서 생성** (Learn 문서 정책 JSON 그대로, `CERT_NAME` 변수 사용; RBAC 전파로 첫 시도 403 가능 → 1–2분 후 재시도 안내를 본문에 명시):

   🟢 실행

   ```bash
   cat > cert-policy.json <<EOF
   {
     "issuerParameters": { "name": "Self" },
     "keyProperties": { "exportable": true, "keyType": "RSA", "keySize": 2048, "reuseKey": false },
     "secretProperties": { "contentType": "application/x-pkcs12" },
     "x509CertificateProperties": {
       "subject": "CN=*.${ZONE_NAME}",
       "subjectAlternativeNames": { "dnsNames": ["*.${ZONE_NAME}", "${ZONE_NAME}"] },
       "validityInMonths": 12,
       "keyUsage": ["digitalSignature", "keyEncipherment"]
     }
   }
   EOF

   az keyvault certificate create \
     --vault-name $KV_NAME \
     --name $CERT_NAME \
     --policy @cert-policy.json

   export CERT_URI=$(az keyvault certificate show \
     --vault-name $KV_NAME \
     --name $CERT_NAME \
     --query id -o tsv | sed 's|/[^/]*$||')
   echo "Cert URI: $CERT_URI"
   ```

   본문: 무버전 URI를 쓰는 이유(인증서 교체 시 operator가 새 버전 자동 감지). `sed 's|/[^/]*$||'`는 Learn 문서 원문 그대로이며 패턴에 변수 보간이 없어 `|` 구분자가 안전함을 주석으로 언급.
6. **UAMI 생성 + 역할 할당 2건** (Learn 문서 그대로):

   🟢 실행

   ```bash
   az identity create --resource-group $RESOURCE_GROUP --name $UAMI_NAME --location $LOCATION
   export UAMI_CLIENT_ID=$(az identity show --resource-group $RESOURCE_GROUP --name $UAMI_NAME --query clientId -o tsv)
   export UAMI_PRINCIPAL_ID=$(az identity show --resource-group $RESOURCE_GROUP --name $UAMI_NAME --query principalId -o tsv)

   az role assignment create \
     --assignee-object-id $UAMI_PRINCIPAL_ID \
     --assignee-principal-type ServicePrincipal \
     --role "DNS Zone Contributor" \
     --scope $ZONE_ID

   az role assignment create \
     --assignee-object-id $UAMI_PRINCIPAL_ID \
     --assignee-principal-type ServicePrincipal \
     --role "Key Vault Secrets User" \
     --scope $(az keyvault show --name $KV_NAME --query id -o tsv)
   ```

7. **ServiceAccount + FIC** (단일 네임스페이스 — Learn 문서의 for 루프를 풀어서):

   🟢 실행

   ```bash
   export OIDC_ISSUER=$(az aks show --resource-group $RESOURCE_GROUP --name $CLUSTER --query oidcIssuerProfile.issuerUrl -o tsv)

   az identity federated-credential create \
     --identity-name $UAMI_NAME \
     --resource-group $RESOURCE_GROUP \
     --name approuting-ws-fic \
     --issuer $OIDC_ISSUER \
     --subject "system:serviceaccount:$APP_NAMESPACE:$SA_NAME" \
     --audiences "api://AzureADTokenExchange"

   kubectl apply -n $APP_NAMESPACE -f - <<EOF
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: $SA_NAME
     annotations:
       azure.workload.identity/client-id: $UAMI_CLIENT_ID
     labels:
       azure.workload.identity/use: "true"
   EOF
   ```

   본문: annotation(SA↔UAMI 연결)과 label(webhook의 토큰 프로젝션 지시)의 역할 각각 설명.
8. **파생 변수 저장**:

   🟢 실행

   ```bash
   cat >> ~/.approuting-ws-env <<EOF
   export ZONE_ID=$ZONE_ID
   export CERT_URI=$CERT_URI
   export UAMI_CLIENT_ID=$UAMI_CLIENT_ID
   EOF
   ```

9. **⏳ 기다리는 동안 읽기**(RBAC 전파 최대 ~5분): ① workload identity·FIC 개념(파드의 SA 토큰 ↔ Entra 토큰 교환, 시크릿 없는 인증), ② Application Routing operator의 reconcile 흐름 — Gateway annotation → SecretProviderClass 생성 → CSI 드라이버가 KV 인증서를 `kubernetes.io/tls` Secret으로 동기화 → listener `certificateRefs` 자동 설정. 👁️ 예시 mermaid 시퀀스 다이어그램 1개.
10. 트러블슈팅 표: certificate create 403(Certificates Officer 전파 지연 → 1–2분 후 재시도) / `az ad signed-in-user show` 실패(게스트 계정) / FIC subject 오타로 이후 모듈에서 인증 실패(진단: `az identity federated-credential list`) 3행.
11. 네비게이션: 이전 03 / 다음 05.

- [ ] **Step 2: 정적 검사**

Run: `cd ~/general/ms-aks-approuting-workshop01 && test $(( $(grep -c '^```' docs/04-dns-tls-infra.md) % 2 )) -eq 0 && grep -c '트러블슈팅' docs/04-dns-tls-infra.md && echo OK`
Expected: 마지막 줄 `OK`

- [ ] **Step 3: Commit**

```bash
git add docs/04-dns-tls-infra.md
git commit -m "docs(04): DNS·TLS 인프라 준비 모듈 작성

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

---

### Task 5: 모듈 05 (TLS Gateway + ClusterExternalDNS) + 매니페스트 2종

**Files:**
- Create: `manifests/gateway-tls.yaml`
- Create: `manifests/cluster-external-dns.yaml`
- Create: `docs/05-tls-gateway-externaldns.md`

**Interfaces:**
- Consumes: Task 3의 `httpbin-gateway`/HTTPRoute `httpbin`(덮어쓰기), Task 4의 `$CERT_URI`·`$ZONE_ID`·`$SA_NAME`·SA/FIC. 0단계는 `source ~/.approuting-ws-env`로 Task 4 파생 변수까지 복원.
- Produces: HTTPS Gateway + A 레코드 + end-to-end HTTPS 검증 절차. 매니페스트는 `__PLACEHOLDER__` 치환 방식(`envsubst` 사용).

- [ ] **Step 1: manifests/gateway-tls.yaml 작성**

`envsubst`용 변수 포함(Learn 문서의 TLS Gateway 예제를 단일 Gateway·`httpbin.<zone>` 호스트로 치환; Task 3과 동일 이름 `httpbin-gateway`로 덮어쓰기, HTTP 리스너 유지 + HTTPS 추가):

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  gatewayClassName: approuting-istio
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: Same
  - name: https
    hostname: httpbin.${ZONE_NAME}
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      options:
        kubernetes.azure.com/tls-cert-keyvault-uri: ${CERT_URI}
        kubernetes.azure.com/tls-cert-service-account: ${SA_NAME}
    allowedRoutes:
      namespaces:
        from: Same
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: httpbin
spec:
  parentRefs:
  - name: httpbin-gateway
  hostnames: ["httpbin.${ZONE_NAME}", "httpbin.example.com"]
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /get
    backendRefs:
    - name: httpbin
      port: 8000
```

- [ ] **Step 2: manifests/cluster-external-dns.yaml 작성**

```yaml
apiVersion: approuting.kubernetes.azure.com/v1alpha1
kind: ClusterExternalDNS
metadata:
  name: workshop-cluster-dns
spec:
  resourceName: workshop-cluster-dns
  resourceNamespace: ${APP_NAMESPACE}
  dnsZoneResourceIDs:
  - ${ZONE_ID}
  resourceTypes:
  - gateway
  identity:
    type: workloadIdentity
    serviceAccount: ${SA_NAME}
```

- [ ] **Step 3: docs/05-tls-gateway-externaldns.md 작성**

구성:

1. H1 `# 05 — TLS Gateway와 ClusterExternalDNS` + 범례 + 예상 시간 `15–20분 (인증서 동기화·A 레코드 생성 대기 포함)`.
2. 0단계 블록(`source` + get-credentials + `echo "CERT_URI=$CERT_URI"` 확인).
3. **TLS Gateway 적용** — 👁️ 예시로 YAML을 보여주며 `tls.options`의 두 키(keyvault-uri, service-account) 설명, 그 후:

   🟢 실행

   ```bash
   cd ~/ms-aks-approuting-workshop01
   envsubst < manifests/gateway-tls.yaml | kubectl apply -n $APP_NAMESPACE -f -
   kubectl wait -n $APP_NAMESPACE --for=condition=programmed gateway httpbin-gateway --timeout=300s
   ```

4. **reconcile 관찰**:

   🟢 실행

   ```bash
   kubectl get secretproviderclass,secret -n $APP_NAMESPACE
   kubectl get gateway httpbin-gateway -n $APP_NAMESPACE -o jsonpath='{.spec.listeners[?(@.name=="https")].tls.certificateRefs}' && echo
   ```

   📋 예상 출력: `kv-gw-cert-httpbin-gateway-https` SecretProviderClass와 동명 `kubernetes.io/tls` Secret(DATA 2), certificateRefs에 해당 Secret 참조 (Learn 문서 output 기반).
5. **ClusterExternalDNS 적용**:

   🟢 실행

   ```bash
   envsubst < manifests/cluster-external-dns.yaml | kubectl apply -f -
   kubectl get pods -n $APP_NAMESPACE -l app=workshop-cluster-dns 2>/dev/null; kubectl get clusterexternaldns
   ```

   본문: `resourceNamespace`에 managed external-dns 인스턴스가 배포되며 SA가 그 네임스페이스에 있어야 함을 설명.
6. **A 레코드 확인**(약 1분 대기 안내):

   🟢 실행

   ```bash
   az network dns record-set a list --resource-group $RESOURCE_GROUP --zone-name $ZONE_NAME -o table
   ```

   📋 예상 출력: `httpbin` A 레코드 1건 (TTL 300).
7. **end-to-end HTTPS 검증** (Learn 문서 verify 절 그대로, 호스트만 치환):

   🟢 실행

   ```bash
   NS=$(az network dns zone show --resource-group $RESOURCE_GROUP --name $ZONE_NAME --query 'nameServers[0]' -o tsv | sed 's/\.$//')
   GATEWAY_IP=$(dig +short @${NS} httpbin.${ZONE_NAME} | tail -1)
   echo "NS=$NS / IP=$GATEWAY_IP"
   curl -k -I --resolve "httpbin.${ZONE_NAME}:443:${GATEWAY_IP}" "https://httpbin.${ZONE_NAME}/get"
   ```

   📋 예상 출력: `HTTP/2 200`.
   **교육 포인트 박스**: ① `dig @NS` — 위임 없는 zone은 권한 네임서버에 직접 물어야 하는 이유, ② `-k`가 필요한 이유(자체 서명; CA 서명 인증서면 `--cacert`로 체인 검증), ③ `--resolve` — 로컬 DNS를 우회하는 테스트 기법, 실 운영은 등록기관 NS 위임.
8. 트러블슈팅 표: Secret이 안 생김(SA annotation/label 누락, FIC subject 불일치 — 진단 `kubectl describe secretproviderclass`) / A 레코드 안 생김(external-dns 파드 로그 확인 `kubectl logs -n $APP_NAMESPACE deploy/workshop-cluster-dns`, DNS Zone Contributor 전파) / curl SSL 오류(`-k` 누락) / `envsubst` 결과에 빈 값(0단계 미수행 — `echo $CERT_URI` 확인) 4행.
9. 네비게이션: 이전 04 / 다음 06.

- [ ] **Step 4: 정적 검사 (envsubst 결과가 유효 YAML인지)**

Run: `cd ~/general/ms-aks-approuting-workshop01 && export ZONE_NAME=z.example CERT_URI=https://kv/x SA_NAME=sa APP_NAMESPACE=ws ZONE_ID=/subscriptions/x && envsubst < manifests/gateway-tls.yaml | python3 -c "import yaml,sys; list(yaml.safe_load_all(sys.stdin))" && envsubst < manifests/cluster-external-dns.yaml | python3 -c "import yaml,sys; list(yaml.safe_load_all(sys.stdin))" && test $(( $(grep -c '^```' docs/05-tls-gateway-externaldns.md) % 2 )) -eq 0 && echo OK`
Expected: `OK`

- [ ] **Step 5: Commit**

```bash
git add manifests/gateway-tls.yaml manifests/cluster-external-dns.yaml docs/05-tls-gateway-externaldns.md
git commit -m "docs(05): TLS Gateway·ClusterExternalDNS 모듈 및 매니페스트 작성

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

---

### Task 6: 모듈 06 (정리)

**Files:**
- Create: `docs/06-cleanup.md`

**Interfaces:**
- Consumes: 전 모듈의 리소스(모두 `$RESOURCE_GROUP` 단일 RG).

- [ ] **Step 1: docs/06-cleanup.md 작성**

구성:

1. H1 `# 06 — 정리` + 범례 + 예상 시간 `~5분 (삭제는 백그라운드 진행)`.
2. 0단계 블록(`source`만).
3. 본문: A 레코드·external-dns가 만든 레코드는 zone과 함께 RG 삭제로 제거됨(별도 정리 불필요 — ClusterExternalDNS 삭제가 레코드를 지우지 않는 제한 사항도 함께 언급).
4. RG 삭제:

   🟢 실행

   ```bash
   az group delete --name $RESOURCE_GROUP --yes --no-wait
   az group show --name $RESOURCE_GROUP --query properties.provisioningState -o tsv
   ```

   📋 예상 출력: `Deleting`.
5. **Key Vault soft-delete 정리** — 본문: RG를 지워도 KV는 soft-delete 상태로 90일 보존, 동일 이름 재사용하려면 purge 필요:

   🟢 실행

   ```bash
   az keyvault purge --name $KV_NAME --no-wait 2>/dev/null || echo "삭제 완료 후 다시 시도하세요"
   ```

6. 로컬 흔적 정리:

   🟢 실행

   ```bash
   rm -f ~/.approuting-ws-env cert-policy.json
   kubectl config delete-context $CLUSTER 2>/dev/null || true
   ```

7. 과금 종료 확인: 포털 비용 분석 반영 지연(수 시간) 안내 + `az group exists --name $RESOURCE_GROUP` 재확인.
8. 트러블슈팅 표: RG 삭제가 오래 걸림(정상 — 클러스터 노드 RG 연쇄 삭제) / purge 권한 오류(Purger 역할 필요) / `MC_` 노드 RG가 남아 보임(자동 삭제 대기) 3행.
9. 네비게이션: 이전 05 / 처음으로(README).

- [ ] **Step 2: 정적 검사**

Run: `cd ~/general/ms-aks-approuting-workshop01 && test $(( $(grep -c '^```' docs/06-cleanup.md) % 2 )) -eq 0 && echo OK`
Expected: `OK`

- [ ] **Step 3: Commit**

```bash
git add docs/06-cleanup.md
git commit -m "docs(06): 정리 모듈 작성

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

---

### Task 7: README + 전체 정합성 검사

**Files:**
- Create: `README.md`
- Modify: (검사에서 발견된 불일치가 있으면 해당 `docs/*.md`)

**Interfaces:**
- Consumes: Task 1–6의 모든 문서·파일명·시간 추정치.

- [ ] **Step 1: README.md 작성 (캐노니컬 스켈레톤 순서)**

섹션 순서 고정:

1. H1 `# AKS Application Routing Gateway API 핸즈온 워크숍` + 한 단락 소개(코어 약 1시간 10분–1시간 30분, Cloud Shell 중심, httpbin을 Gateway API로 노출 → Azure DNS·Key Vault TLS까지).
2. **아키텍처** — mermaid:

   ```mermaid
   flowchart LR
     user([사용자]) -->|"HTTP → HTTPS"| gw["Gateway (approuting-istio)<br/>httpbin-gateway"]
     gw --> route[HTTPRoute] --> svc[httpbin Service]
     subgraph aks ["AKS Standard (Korea Central)"]
       gw; route; svc
       edns[managed external-dns]
     end
     edns -.->|A 레코드 발행| zone[(Azure DNS zone)]
     gw -.->|인증서 동기화<br/>SecretProviderClass| kv[(Key Vault<br/>자체 서명 인증서)]
     uami[UAMI + 페더레이션<br/>자격 증명] -.-> edns
     uami -.-> gw
   ```

3. **학습 목표** 번호 목록 6개: ① 필요한 기능(Managed Gateway API CRD·application routing Gateway API·workload identity·KV CSI)을 켠 AKS를 az CLI로 생성, ② Gateway·HTTPRoute로 앱을 HTTP 노출하고 자동 생성 리소스 이해, ③ HTTPRoute의 hostname·path 매칭 동작 관찰, ④ DNS zone·KV 인증서·UAMI·FIC로 operator 통합 인프라 구성, ⑤ Gateway annotation 기반 TLS 종료와 ClusterExternalDNS 레코드 자동 발행 확인, ⑥ 리소스 전체 정리.
4. **사전 요구사항** 표: Azure 구독(Owner 또는 RBAC Administrator + Managed Identity Contributor) / Cloud Shell(bash) / az ≥ 2.86.0 / Kubernetes 기초(파드·서비스 개념).
5. **모듈 목차** 표(01–06, 한 줄 설명, 상대 링크 `docs/0N-...md`).
6. **시간표** — 스펙 8절의 표 그대로(모듈·제목·예상 시간·주 소요 요인, 전체 ≈ 1시간 10분–1시간 30분) + "리허설 후 실측 보정 예정" 각주.
7. **비용 개요**: Standard AKS 노드 2대(Standard_D2s_v5) + Standard LB + DNS zone + Key Vault, 실습 1.5시간 기준 소액, 06 정리 필수.
8. **트러블슈팅 색인** 표: 각 모듈 트러블슈팅 표의 대표 증상 1–2개씩 → 해당 모듈 링크.
9. **참고 자료**: 근거 Learn 문서 3편 링크 + Gateway API 공식 사이트.

- [ ] **Step 2: 전체 정합성 검사 스크립트 실행**

Run:

```bash
cd ~/general/ms-aks-approuting-workshop01
# 1) 코드 펜스 짝수
for f in README.md docs/0*.md; do n=$(grep -c '^```' $f); [ $((n % 2)) -eq 0 ] || echo "ODD-FENCE: $f"; done
# 2) 라벨 규칙: docs 내 여는 펜스 직전 줄에 🟢/👁️/📋 존재 (mermaid 제외)
python3 - <<'PY'
import re, glob
for f in glob.glob('docs/0*.md'):
    lines = open(f).read().splitlines()
    inside = False
    for i, l in enumerate(lines):
        if l.startswith('```'):
            if not inside:
                if 'mermaid' not in l and not any(s in lines[i-1] for s in ('🟢','👁️','📋')):
                    print(f'NO-LABEL: {f}:{i+1}')
            inside = not inside
PY
# 3) 내부 링크 대상 파일 존재
grep -ohE '\]\((docs/)?0[1-6][^)]*\.md' README.md docs/0*.md | tr -d '](' | sort -u | while read p; do [ -f "$p" ] || [ -f "docs/$p" ] || echo "BROKEN-LINK: $p"; done
# 4) 외부 링크 상태 코드
grep -ohE 'https://[^) ]+' README.md docs/0*.md | sort -u | while read u; do c=$(curl -s -o /dev/null -w '%{http_code}' -L --max-time 15 "$u"); [ "$c" = 200 ] || echo "LINK $c: $u"; done
# 5) 물결 범위 표기 검사(취소선 위험): 숫자~숫자 패턴
grep -nE '[0-9]~[0-9]' README.md docs/0*.md || true
echo DONE
```

Expected: `ODD-FENCE`/`NO-LABEL`/`BROKEN-LINK`/`LINK 4xx` 출력 없이 `DONE`. 발견 시 해당 파일 수정 후 재실행.

- [ ] **Step 3: README·모듈 간 시간 추정치 일치 육안 확인**

각 `docs/0N-*.md` 도입부의 예상 시간과 README 시간표의 값이 문자열까지 동일한지 확인. 불일치 시 README 쪽을 모듈 값에 맞춰 수정.

- [ ] **Step 4: Commit**

```bash
git add README.md docs/ manifests/
git commit -m "docs(readme): README 및 전체 정합성 검사 완료

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

---

## 계획 범위 밖 (후속 단계)

- **Azure 실 리허설**: 실제 구독에서 01→06 복붙 1회 수행 → 📋 예상 출력 실측 교체, `<!-- TODO(rehearsal) -->` 주석 제거, 시간표 실측 보정, `--enable-app-routing-istio`가 preview extension을 요구하면 01에 설치 단계 추가. (스펙 7절 — 콘텐츠 작성 완료 후 별도 세션에서.)
- GitHub 저장소 생성·push.
