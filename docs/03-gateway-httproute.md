# 03 — Gateway·HTTPRoute로 HTTP 노출

> 🟢 실행 = 직접 입력·수행 · 👁️ 예시 = 눈으로만(개념/발췌) · 📋 예상 출력 = 비교용(입력 불필요)

예상 소요 시간: 10–15분 (LB IP 할당 대기 포함)

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
NAME                                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/httpbin-gateway-approuting-istio      2/2     2            2           2m

NAME                                         TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)        AGE
service/httpbin-gateway-approuting-istio     LoadBalancer   10.0.X.X      20.XXX.XXX.XXX   80:XXXXX/TCP   2m

NAME                                                                  REFERENCE                                        TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/httpbin-gateway-approuting-istio  Deployment/httpbin-gateway-approuting-istio      <unknown>/50%   2         5         2          2m

NAME                                                         MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
poddisruptionbudget.policy/httpbin-gateway-approuting-istio  1               N/A               1                     2m
```

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
Gateway IP: 20.XXX.XXX.XXX
HTTP/1.1 200 OK
server: istio-envoy
...
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

## 트러블슈팅

| 증상 | 원인 | 해결 방법 |
|------|------|-----------|
| `EXTERNAL-IP`가 `<pending>` 상태로 계속 유지 | Azure Load Balancer 프로비저닝 지연 또는 구독 할당량 부족 | `kubectl describe gateway httpbin-gateway -n $APP_NAMESPACE`로 이벤트를 확인합니다. 수 분 후에도 해결되지 않으면 구독의 공용 IP 할당량을 확인합니다 |
| `kubectl wait` 명령이 타임아웃(`condition not met`) | `gatewayClassName` 오타 또는 application routing 애드온 비활성화 | `kubectl get gateway httpbin-gateway -n $APP_NAMESPACE -o yaml`로 `spec.gatewayClassName` 값이 `approuting-istio`인지 확인합니다. 오타가 있으면 매니페스트를 수정하고 다시 적용합니다 |
| `curl` 요청에서 404 반환 | `Host` 헤더 누락 또는 HTTPRoute `hostnames` 불일치 | `-HHost:httpbin.example.com` 옵션을 추가했는지 확인합니다. `kubectl get httproute httpbin -n $APP_NAMESPACE -o yaml`로 `spec.hostnames` 값이 정확한지 검토합니다 |

---

[← 02 — 환경 준비](02-environment-setup.md) | 다음: [04 — DNS·TLS 인프라 준비](04-dns-tls-infra.md)

<!-- TODO(rehearsal): 예상 출력 실측 검증 -->
