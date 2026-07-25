# 03 — Gateway·HTTPRoute로 HTTP 노출

> 🟢 실행 = 직접 입력·수행 · 👁️ 예시 = 눈으로만(개념/발췌) · 📋 예상 출력 = 비교용(입력 불필요)

예상 소요 시간: 10–15분 (LB IP 할당 대기 포함, 8절 옵션 수행 시 +5분)

---

<details>
<summary>🔄 0단계 — 변수 재설정 (새 터미널/세션에서 시작하는 경우)</summary>

🟢 **실행**
```bash
source ~/.approuting-ws-env
echo "RESOURCE_GROUP=$RESOURCE_GROUP"
```

🟢 **실행**
```bash
az aks get-credentials --resource-group $RESOURCE_GROUP --name $CLUSTER --overwrite-existing || true
```

</details>

---

## 1. 워크샵 리포지토리 클론

이 모듈부터 `manifests/` 디렉터리의 YAML 파일을 사용합니다. 리포지토리가 없으면 클론하고, 이미 있으면 최신 상태로 업데이트합니다.

🟢 **실행**
```bash
cd ~ && git clone https://github.com/jungwoonlee_microsoft/ms-aks-approuting-workshop01.git 2>/dev/null || (cd ms-aks-approuting-workshop01 && git pull)
cd ~/ms-aks-approuting-workshop01
```

---

## 2. 네임스페이스 생성 및 httpbin 배포

`workshop` 네임스페이스를 멱등(idempotent) 방식으로 생성한 뒤, Istio 공식 샘플인 `httpbin`을 배포합니다.
`httpbin`은 HTTP 요청의 헤더·파라미터·응답 코드를 그대로 반환하는 테스트용 서비스로, Gateway API 동작을 확인하기에 적합합니다.

🟢 **실행**
```bash
kubectl create namespace $APP_NAMESPACE --dry-run=client -o yaml | kubectl apply -f -
export ISTIO_RELEASE="release-1.27"
kubectl apply -n $APP_NAMESPACE -f https://raw.githubusercontent.com/istio/istio/$ISTIO_RELEASE/samples/httpbin/httpbin.yaml
kubectl get pods -n $APP_NAMESPACE
```

파드가 `Running` 상태가 되면 다음 단계로 진행합니다.

---

## 3. Gateway·HTTPRoute 매니페스트 내용 확인

클러스터에 적용하기 전에 매니페스트의 각 필드를 살펴봅니다.

👁️ **예시**
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  gatewayClassName: approuting-istio   # AKS application routing 애드온이 관리하는 GatewayClass
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: Same                     # 동일 네임스페이스의 HTTPRoute만 허용
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: httpbin
spec:
  parentRefs:
  - name: httpbin-gateway              # 위에서 선언한 Gateway에 연결
  hostnames: ["httpbin.example.com"]   # 이 호스트 헤더를 가진 요청만 처리
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /get                    # /get 이하 경로만 매칭
    backendRefs:
    - name: httpbin
      port: 8000
```

**주요 필드 설명:**

| 필드 | 역할 |
|------|------|
| `gatewayClassName: approuting-istio` | AKS application routing 애드온이 관리하는 GatewayClass를 지정합니다. 이 값이 틀리면 Gateway가 생성되지 않습니다. |
| `listeners[].allowedRoutes.namespaces.from: Same` | 동일 네임스페이스에 있는 HTTPRoute만 이 Gateway에 연결할 수 있습니다. |
| `parentRefs[].name` | HTTPRoute가 연결될 Gateway 이름입니다. |
| `hostnames` | 이 HTTPRoute가 처리할 가상 호스트 목록입니다. Host 헤더가 일치하지 않으면 404를 반환합니다. |
| `rules[].matches[].path` | 경로 프리픽스 매칭 조건입니다. `/get` 이외의 경로는 이 HTTPRoute가 처리하지 않습니다. |

---

## 4. Gateway·HTTPRoute 적용 및 프로그래밍 대기

🟢 **실행**
```bash
kubectl apply -n $APP_NAMESPACE -f manifests/gateway-http.yaml
kubectl wait -n $APP_NAMESPACE --for=condition=programmed gateway httpbin-gateway --timeout=300s
```

`kubectl wait` 명령은 Gateway가 `Programmed` 상태, 즉 Azure Load Balancer가 실제로 프로비저닝되어 외부 IP가 할당될 때까지 최대 5분 대기합니다.

---

## 5. 자동 생성 리소스 확인

`Gateway`가 `Programmed` 상태가 되면 application routing 컨트롤러는 다음 리소스를 자동으로 생성합니다.
리소스 이름 규칙은 **`<Gateway 이름>-<GatewayClass 이름>`** 입니다.
이 워크샵에서는 `httpbin-gateway`(Gateway 이름)와 `approuting-istio`(GatewayClass 이름)를 조합하여 `httpbin-gateway-approuting-istio`라는 이름이 사용됩니다.

🟢 **실행**
```bash
kubectl get deployment,service,hpa,pdb -n $APP_NAMESPACE httpbin-gateway-approuting-istio
```

📋 **예상 출력**
```
NAME                                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/httpbin-gateway-approuting-istio   2/2     2            2           3m55s

NAME                                       TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)                        AGE
service/httpbin-gateway-approuting-istio   LoadBalancer   10.0.132.62   20.249.51.10   15021:31390/TCP,80:30809/TCP   3m55s

NAME                                                                   REFERENCE                                     TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/httpbin-gateway-approuting-istio   Deployment/httpbin-gateway-approuting-istio   cpu: 2%/80%   2         5         2          3m56s

NAME                                                          MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
poddisruptionbudget.policy/httpbin-gateway-approuting-istio   1               N/A               1                     3m56s
```

> **참고** Service의 `PORT(S)`에는 80 외에 Istio 상태 점검용 포트 `15021`이 함께 노출됩니다. HPA `TARGETS`는 metrics-server 수집 직후 일시적으로 `<unknown>/80%`로 보일 수 있습니다.

> **참고** `EXTERNAL-IP`에 실제 IP가 표시되면 다음 단계로 진행합니다. `<pending>` 상태가 지속되면 아래 트러블슈팅 표를 참고합니다.

---

## 6. HTTP 요청으로 동작 확인

Gateway의 외부 IP를 가져와 `Host` 헤더를 지정하여 HTTP 요청을 보냅니다.

🟢 **실행**
```bash
export INGRESS_HOST=$(kubectl get gateway httpbin-gateway -n $APP_NAMESPACE -ojsonpath='{.status.addresses[0].value}')
echo "Gateway IP: $INGRESS_HOST"
curl -s -I -HHost:httpbin.example.com "http://$INGRESS_HOST/get"
```

📋 **예상 출력**
```
Gateway IP: 20.249.51.10
HTTP/1.1 200 OK
access-control-allow-credentials: true
access-control-allow-origin: *
content-type: application/json; charset=utf-8
date: Sat, 25 Jul 2026 06:17:41 GMT
x-envoy-upstream-service-time: 0
server: istio-envoy
transfer-encoding: chunked
```

`HTTP/1.1 200 OK`가 반환되면 Gateway와 HTTPRoute가 올바르게 동작하고 있는 것입니다.

---

## 7. 실험 — HTTPRoute 매칭 동작 체감

Gateway API의 핵심 개념인 **호스트 매칭**과 **경로 매칭**이 실제로 어떻게 작동하는지 확인합니다.

- HTTPRoute에 `hostnames: ["httpbin.example.com"]`을 선언했으므로, `Host` 헤더가 정확히 일치해야만 요청이 처리됩니다.
- 경로 매칭 규칙은 `/get` 프리픽스만 허용하므로, `/headers` 같은 다른 경로는 이 HTTPRoute가 처리하지 않습니다.
- 매칭에 실패한 요청은 **404**를 반환합니다. 이는 오류가 아니라 Gateway API의 의도된 보안 동작입니다.

🟢 **실행**
```bash
curl -s -o /dev/null -w "Host 불일치: %{http_code}\n" "http://$INGRESS_HOST/get"
curl -s -o /dev/null -w "경로 불일치: %{http_code}\n" -HHost:httpbin.example.com "http://$INGRESS_HOST/headers"
```

📋 **예상 출력**
```
Host 불일치: 404
경로 불일치: 404
```

두 요청 모두 404를 반환하는 것이 정상입니다. `Host` 헤더 없이 IP로 직접 접근하면 어떤 HTTPRoute도 매칭되지 않고, `/headers` 경로는 이 HTTPRoute의 규칙에 해당하지 않기 때문입니다.

---

## 8. 옵션 — Gateway 외부 IP 고정 (정적 공인 IP)

기본 동작에서는 Azure Load Balancer가 **동적 공인 IP**를 할당하므로, Gateway를 삭제 후 재생성하면 IP가 바뀝니다.
방화벽 허용 목록 등록이나 외부 DNS 수동 등록처럼 IP가 바뀌면 안 되는 환경에서는 **정적 공인 IP**를 만들어 Gateway에 고정할 수 있습니다.

Gateway API 표준 필드인 `spec.addresses`에 IP를 선언하면, Istio 컨트롤러가 자동 생성하는 LoadBalancer Service에 해당 IP를 지정합니다.

### 8.1 노드 리소스 그룹에 정적 공인 IP 생성

AKS가 관리하는 **노드 리소스 그룹**(`MC_...`)에 IP를 생성합니다. 클러스터의 kubelet identity가 이 RG에 이미 네트워크 권한을 갖고 있어 별도 역할 할당이 필요 없고, 클러스터 삭제 시 IP도 함께 정리됩니다.

🟢 **실행**
```bash
export NODE_RG=$(az aks show --resource-group $RESOURCE_GROUP --name $CLUSTER --query nodeResourceGroup -o tsv)
az network public-ip create \
  --resource-group $NODE_RG \
  --name pip-httpbin-gateway \
  --sku Standard \
  --allocation-method Static \
  --location $LOCATION \
  --query 'publicIp.ipAddress' -o tsv
export STATIC_IP=$(az network public-ip show --resource-group $NODE_RG --name pip-httpbin-gateway --query ipAddress -o tsv)
echo "STATIC_IP=$STATIC_IP"
```

📋 **예상 출력**
```
20.196.222.78
STATIC_IP=20.196.222.78
```

> **참고** 실 운영에서 IP를 클러스터 수명과 분리하려면 별도 RG에 IP를 만들고, 그 RG에 대해 클러스터 identity에 `Network Contributor` 역할을 할당해야 합니다. 이 워크샵에서는 정리 단순화를 위해 노드 RG를 사용합니다.

### 8.2 Gateway에 `spec.addresses` 지정

기존 Gateway를 patch하여 정적 IP를 선언합니다.

🟢 **실행**
```bash
kubectl patch gateway httpbin-gateway -n $APP_NAMESPACE --type merge \
  -p "{\"spec\":{\"addresses\":[{\"type\":\"IPAddress\",\"value\":\"$STATIC_IP\"}]}}"
kubectl wait -n $APP_NAMESPACE --for=condition=programmed gateway httpbin-gateway --timeout=300s
```

👁️ **예시** — 매니페스트로 처음부터 선언하는 경우 Gateway 전체 YAML은 다음과 같습니다 (`manifests/gateway-http.yaml`의 Gateway에 `addresses` 필드만 추가된 형태).
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  gatewayClassName: approuting-istio
  addresses:
  - type: IPAddress
    value: 20.196.222.78   # 미리 생성한 정적 공인 IP
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: Same
```

### 8.3 고정 IP 확인

Service의 `EXTERNAL-IP`와 Gateway 상태 주소가 모두 정적 IP로 교체되고, 동일하게 200이 반환되는지 확인합니다.
Load Balancer 프런트엔드 재구성에 최대 1–2분이 걸릴 수 있습니다.

🟢 **실행**
```bash
kubectl get service httpbin-gateway-approuting-istio -n $APP_NAMESPACE
kubectl get gateway httpbin-gateway -n $APP_NAMESPACE -ojsonpath='{.status.addresses[0].value}'; echo
curl -s -o /dev/null -w "고정 IP 응답: %{http_code}\n" -HHost:httpbin.example.com "http://$STATIC_IP/get"
```

📋 **예상 출력**
```
NAME                               TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)                        AGE
httpbin-gateway-approuting-istio   LoadBalancer   10.0.100.97   20.196.222.78   15021:30286/TCP,80:30498/TCP   85s
20.196.222.78
고정 IP 응답: 200
```

이제 Gateway를 삭제하고 다시 만들어도 `spec.addresses`만 동일하게 선언하면 항상 같은 IP로 노출됩니다.

> **주의** Gateway를 재생성하는 경우 `kubectl apply` 직후 patch를 다시 적용해야 합니다. `manifests/gateway-http.yaml`에는 `addresses`가 없으므로, patch 없이 재생성하면 동적 IP로 돌아갑니다. Gateway `status.addresses`가 일시적으로 이전 동적 IP를 표시하더라도 1분 내에 정적 IP로 수렴합니다.
>
> **참고** 04–05 모듈(DNS·TLS)은 고정 IP 없이도 동작합니다. ExternalDNS가 Gateway의 현재 IP를 감지해 A 레코드를 자동 갱신하기 때문입니다. 반대로 이 옵션을 수행했다면 05 모듈에서 ExternalDNS 없이 [수동 A 레코드 경로(05 모듈 6절)](05-tls-gateway-externaldns.md#6-옵션--정적-ip--수동-a-레코드-externaldns-생략)를 선택할 수 있습니다.

---

## 트러블슈팅

| 증상 | 원인 | 해결 방법 |
|------|------|-----------|
| `EXTERNAL-IP`가 `<pending>` 상태로 계속 유지 | Azure Load Balancer 프로비저닝 지연 또는 구독 할당량 부족 | `kubectl describe gateway httpbin-gateway -n $APP_NAMESPACE`로 이벤트를 확인합니다. 수 분 후에도 해결되지 않으면 구독의 공용 IP 할당량을 확인합니다 |
| `kubectl wait` 명령이 타임아웃(`condition not met`) | `gatewayClassName` 오타 또는 application routing 애드온 비활성화 | `kubectl get gateway httpbin-gateway -n $APP_NAMESPACE -o yaml`로 `spec.gatewayClassName` 값이 `approuting-istio`인지 확인합니다. 오타가 있으면 매니페스트를 수정하고 다시 적용합니다 |
| `curl` 요청에서 404 반환 | `Host` 헤더 누락 또는 HTTPRoute `hostnames` 불일치 | `-HHost:httpbin.example.com` 옵션을 추가했는지 확인합니다. `kubectl get httproute httpbin -n $APP_NAMESPACE -o yaml`로 `spec.hostnames` 값이 정확한지 검토합니다 |
| (8절) `spec.addresses` 지정 후 `EXTERNAL-IP`가 `<pending>`으로 남음 | 정적 IP의 SKU가 `Basic`이거나, IP가 노드 RG 외부에 있어 클러스터 identity에 권한이 없음 | `az network public-ip show --resource-group $NODE_RG --name pip-httpbin-gateway --query sku.name -o tsv`로 `Standard`인지 확인합니다. 노드 RG 외부의 IP라면 해당 RG에 클러스터 identity의 `Network Contributor` 역할을 할당하고 `kubectl describe svc httpbin-gateway-approuting-istio -n $APP_NAMESPACE` 이벤트를 확인합니다 |

---

[← 02 — 환경 준비](02-environment-setup.md) | 다음: [04 — DNS·TLS 인프라 준비](04-dns-tls-infra.md)
