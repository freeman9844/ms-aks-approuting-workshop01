# 06 — Gateway 인프라 커스터마이징 (옵션)

> 🟢 실행 = 직접 입력·수행 · 👁️ 예시 = 눈으로만(개념/발췌) · 📋 예상 출력 = 비교용(입력 불필요)

예상 소요 시간: 10–15분

> **옵션 모듈**: 이 모듈은 선택 사항입니다. 건너뛰고 [07 — AFD 카나리 마이그레이션 (옵션)](07-afd-canary-migration.md) 또는 [08 — 정리](08-cleanup.md)로 이동해도 됩니다.

Gateway 리소스를 만들면 application routing add-on(관리형 Istio)이 뒤에서 LoadBalancer **Service**, **Deployment**, **HPA**(HorizontalPodAutoscaler), **PDB**(PodDisruptionBudget)를 자동으로 생성합니다. 03 모듈에서 이미 `httpbin-gateway-approuting-istio`라는 이름으로 이 리소스들을 확인했습니다.

이 모듈에서는 자동 생성된 인프라 리소스를 커스터마이징하는 두 가지 방법을 실습합니다.

1. **ConfigMap customizations** — GatewayClass 기본값 확인 및 Gateway별 ConfigMap으로 HPA·Deployment 재정의
2. **Annotation customizations** — `spec.infrastructure.annotations`로 내부(Internal) Load Balancer 전환을 실제 수행

> ⚠️ 마지막 절(3절)의 내부 LB 전환을 수행하면 03 모듈에서 고정한 정적 공인 IP 기반의 **외부 접근이 끊어집니다**. 끊어진 상태 그대로 [08 — 정리](08-cleanup.md)로 진행하면 됩니다. 옵션 [07](07-afd-canary-migration.md)을 이어서 진행하려면 07 모듈 상단의 복원 절차를 따르세요.

참고: [Gateway API 지원 구성 — ConfigMap customizations](https://learn.microsoft.com/en-us/azure/aks/istio-gateway-api#configmap-customizations)

---

<details>
<summary>🔄 0단계 — 변수 재설정 (새 터미널/세션에서 시작하는 경우)</summary>

🟢 **실행**
```bash
source ~/.approuting-ws-env
echo "STATIC_IP=$STATIC_IP  ZONE_NAME=$ZONE_NAME"
```

</details>

---

## 1. GatewayClass 수준 기본값 확인

Gateway API용 관리형 CRD와 Istio 기반 add-on이 활성화되면, AKS는 `aks-istio-system` 네임스페이스에 GatewayClass 수준 기본값을 담은 ConfigMap `istio-gateway-class-defaults`를 자동으로 배포합니다(클러스터 생성 후 나타나기까지 최대 5분 소요될 수 있음).

🟢 **실행**
```bash
# GatewayClass 기본값 ConfigMap의 데이터와 라벨 확인
kubectl get configmap istio-gateway-class-defaults -n aks-istio-system -o yaml | head -12
kubectl get configmap istio-gateway-class-defaults -n aks-istio-system \
  -o jsonpath='{.metadata.labels.gateway\.istio\.io/defaults-for-class}'; echo
```

📋 **예상 출력**
```
apiVersion: v1
data:
  horizontalPodAutoscaler: |
    spec:
      minReplicas: 2
      maxReplicas: 5
  podDisruptionBudget: |
    spec:
      minAvailable: 1
kind: ConfigMap
metadata:
  annotations:
approuting-istio
```

`gateway.istio.io/defaults-for-class: approuting-istio` 라벨이 이 ConfigMap을 `approuting-istio` GatewayClass의 기본값으로 지정합니다. 03 모듈에서 확인했던 HPA의 `MINPODS 2 / MAXPODS 5`가 바로 이 기본값에서 온 것입니다.

🟢 **실행**
```bash
# 현재 Gateway의 HPA — GatewayClass 기본값(2/5)이 적용된 상태
kubectl get hpa httpbin-gateway-approuting-istio -n workshop
```

📋 **예상 출력**
```
NAME                               REFERENCE                                     TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
httpbin-gateway-approuting-istio   Deployment/httpbin-gateway-approuting-istio   cpu: 2%/80%   2         5         2          34m
```

---

## 2. Gateway별 커스터마이징 — parametersRef

특정 Gateway에만 다른 설정을 적용하려면 같은 네임스페이스에 ConfigMap을 만들고 Gateway의 `spec.infrastructure.parametersRef`로 참조합니다. Gateway 수준 설정이 GatewayClass 수준 기본값보다 우선합니다.

HPA 레플리카 범위를 3–4로 바꾸고 Deployment에 라벨을 추가해 보겠습니다.

🟢 **실행**
```bash
# 1) Gateway별 커스터마이징 ConfigMap 생성
kubectl apply -n workshop -f - <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: gw-options
data:
  horizontalPodAutoscaler: |
    spec:
      minReplicas: 3
      maxReplicas: 4
  deployment: |
    metadata:
      labels:
        workshop.example.com/gateway-config: per-gateway
EOF

# 2) Gateway가 ConfigMap을 참조하도록 parametersRef 추가
kubectl patch gateway httpbin-gateway -n workshop --type merge \
  -p '{"spec":{"infrastructure":{"parametersRef":{"group":"","kind":"ConfigMap","name":"gw-options"}}}}'
```

📋 **예상 출력**
```
configmap/gw-options created
gateway.gateway.networking.k8s.io/httpbin-gateway patched
```

약 30초 후 변경 사항이 반영됩니다.

🟢 **실행**
```bash
# HPA가 3/4로 바뀌고 레플리카가 3개로 늘었는지 확인
kubectl get hpa httpbin-gateway-approuting-istio -n workshop

# Deployment에 커스텀 라벨이 주입되었는지 확인
kubectl get deployment httpbin-gateway-approuting-istio -n workshop \
  -o jsonpath='{.metadata.labels.workshop\.example\.com/gateway-config}'; echo
```

📋 **예상 출력**
```
NAME                               REFERENCE                                     TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
httpbin-gateway-approuting-istio   Deployment/httpbin-gateway-approuting-istio   cpu: 2%/80%   3         4         3          34m
per-gateway
```

커스터마이징 가능한 필드는 리소스별 허용 목록(allow list)으로 제한됩니다. 예를 들어 HPA의 `minReplicas`는 2 이상이어야 하며, 허용되지 않은 값은 검증 웹훅이 즉시 거부합니다(하단 트러블슈팅 참고).

🟢 **실행**
```bash
# 정적 공인 IP 기반 외부 접근이 그대로 유지되는지 확인
curl -s -o /dev/null -w "%{http_code}\n" http://$STATIC_IP/get -H "Host: httpbin.$ZONE_NAME"
```

📋 **예상 출력**
```
200
```

> 커스터마이징한 HPA(3/4)와 ConfigMap `gw-options`는 별도로 원복하지 않아도 됩니다. [08 — 정리](08-cleanup.md)에서 리소스 그룹과 클러스터가 통째로 삭제됩니다.

---

## 3. Annotation customizations — 내부 Load Balancer 전환

Gateway의 `spec.infrastructure.annotations`에 지정한 어노테이션은 자동 생성되는 LoadBalancer Service에 그대로 전달됩니다. 대표적인 활용 예는 **내부(Internal) Load Balancer** 전환으로, VNet 내부에서만 접근 가능한 프라이빗 게이트웨이를 만들 수 있습니다.

> ⚠️ **이 실습을 수행하면 정적 공인 IP 기반의 외부 접근이 끊어집니다.** 완료 후 바로 [08 — 정리](08-cleanup.md)로 이동하면 되며, 옵션 [07](07-afd-canary-migration.md)을 진행할 계획이라면 07 모듈 상단의 복원 절차를 따르세요.

### 3.1 내부 LB 어노테이션 추가 — 정적 공인 IP와의 충돌 관찰

먼저 현재 Gateway(정적 공인 IP가 `spec.addresses`에 고정된 상태)에 내부 LB 어노테이션만 추가해 봅니다.

🟢 **실행**
```bash
# 내부 LB 어노테이션 추가 (spec.addresses의 공인 IP는 아직 그대로)
kubectl patch gateway httpbin-gateway -n workshop --type merge \
  -p '{"spec":{"infrastructure":{"annotations":{"service.beta.kubernetes.io/azure-load-balancer-internal":"true"}}}}'

# 약 30초 후 Service 이벤트 확인
sleep 30
kubectl get events -n workshop \
  --field-selector involvedObject.name=httpbin-gateway-approuting-istio,reason=SyncLoadBalancerFailed \
  -o jsonpath='{.items[-1].message}' | tail -c 300; echo
```

👁️ **예시** — patch 적용 후 Gateway 전체 YAML은 다음과 같습니다 (05 모듈의 `gateway-tls.yaml`에 `infrastructure.annotations`가 추가된 형태 — `spec.addresses`의 공인 IP와 2절의 `parametersRef`는 그대로).
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: httpbin-gateway
  namespace: workshop
spec:
  gatewayClassName: approuting-istio
  addresses:
  - type: IPAddress
    value: 20.196.222.78   # 03 모듈에서 고정한 정적 공인 IP
  infrastructure:
    annotations:
      service.beta.kubernetes.io/azure-load-balancer-internal: "true"   # 이번에 추가된 어노테이션
    parametersRef:          # 2절에서 추가한 Gateway별 커스터마이징
      group: ""
      kind: ConfigMap
      name: gw-options
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
```

📋 **예상 출력**
```
{
  "error": {
    "code": "PrivateIPAddressNotInSubnet",
    "message": "Private static IP address 20.214.170.40 does not belong to the range of subnet prefix 10.224.0.0/16.",
    "details": []
  }
}
```

의도된 실패입니다 — 내부 LB는 노드 서브넷(`10.224.0.0/16`)의 사설 IP만 사용할 수 있는데, `spec.addresses`에 고정된 **공인** IP를 사설 프런트엔드로 쓰려고 해서 Azure가 거부한 것입니다. 어노테이션은 이렇게 Service를 거쳐 Azure LB 구성까지 그대로 전달된다는 것을 확인할 수 있습니다.

### 3.2 spec.addresses 제거 — 내부 IP 자동 할당

공인 IP 고정을 제거하면 내부 LB가 노드 서브넷에서 사설 IP를 자동 할당받습니다.

🟢 **실행**
```bash
# 정적 공인 IP 고정 제거
kubectl patch gateway httpbin-gateway -n workshop --type json \
  -p '[{"op":"remove","path":"/spec/addresses"}]'

# 약 45초 후 Service와 Gateway의 주소 확인
sleep 45
kubectl get svc httpbin-gateway-approuting-istio -n workshop
kubectl get gateway httpbin-gateway -n workshop
```

📋 **예상 출력**
```
NAME                               TYPE           CLUSTER-IP   EXTERNAL-IP   PORT(S)                                      AGE
httpbin-gateway-approuting-istio   LoadBalancer   10.0.132.1   10.224.0.6    15021:30368/TCP,80:32682/TCP,443:32711/TCP   65m
NAME              CLASS              ADDRESS      PROGRAMMED   AGE
httpbin-gateway   approuting-istio   10.224.0.6   True         65m
```

`EXTERNAL-IP`가 공인 IP 대신 노드 서브넷의 **사설 IP**(`10.224.x.x`)로 바뀌었습니다.

### 3.3 접근 경로 검증 — 외부 단절, 내부 정상

🟢 **실행**
```bash
# 1) 외부(Cloud Shell)에서 기존 정적 공인 IP로 접근 — 더 이상 응답하지 않음
curl -s -o /dev/null -w "%{http_code}\n" --max-time 10 http://$STATIC_IP/get -H "Host: httpbin.$ZONE_NAME"

# 2) 클러스터 내부 파드에서 사설 IP로 접근 — 정상 응답
INTERNAL_IP=$(kubectl get svc httpbin-gateway-approuting-istio -n workshop -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
kubectl run curl-test -n workshop --rm -i --restart=Never --image=curlimages/curl:8.10.1 -- \
  curl -s -o /dev/null -w "%{http_code}\n" --max-time 10 http://$INTERNAL_IP/get -H "Host: httpbin.$ZONE_NAME"
```

📋 **예상 출력**
```
000
200
pod "curl-test" deleted from workshop namespace
```

외부 요청은 타임아웃(`000`)되고, VNet 내부(파드)에서는 정상 응답(`200`)합니다. 게이트웨이가 내부 전용으로 전환된 것입니다.

### 3.4 참고 — public Gateway와 private Gateway 병행 운영

실 운영에서는 기존 public Gateway를 내부용으로 전환하는 대신, **별도의 private Gateway를 하나 더 두는** 구성이 일반적입니다. Gateway마다 독립적인 LoadBalancer Service·Deployment·HPA·PDB가 생성되므로(`<gateway 이름>-approuting-istio`) 두 Gateway는 서로 간섭하지 않으며, 같은 백엔드를 HTTPRoute의 `parentRefs`로 양쪽에 연결할 수도 있습니다.

👁️ **예시** — public `httpbin-gateway` 옆에 추가하는 private Gateway (이 워크샵에서는 적용하지 않습니다)
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: internal-gateway
  namespace: workshop
spec:
  gatewayClassName: approuting-istio
  infrastructure:
    annotations:
      service.beta.kubernetes.io/azure-load-balancer-internal: "true"
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
  name: httpbin-internal
  namespace: workshop
spec:
  parentRefs:
  - name: internal-gateway
  hostnames: ["httpbin.internal.example"]
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /get
    backendRefs:
    - name: httpbin
      port: 8000
```

> **이 워크샵 클러스터에서 실습하지 않는 이유**: Gateway 하나당 파드 2개(HPA `minReplicas: 2`, 각 CPU 100m 요청)가 추가로 필요합니다. 이 워크샵의 2노드 클러스터는 CPU 요청량이 이미 거의 가득 차 있어(노드당 약 98–99%), 새 Gateway 파드가 `Insufficient cpu`로 `Pending`에 머물고 Gateway가 `PROGRAMMED: False` 상태로 남습니다. 기능의 제약이 아니라 노드 리소스의 제약이므로, 노드 수를 늘리거나 더 큰 VM SKU를 사용하면 정상 동작합니다.

이것으로 이 모듈의 실습이 끝났습니다. [07 — AFD 카나리 마이그레이션 (옵션)](07-afd-canary-migration.md)을 이어서 진행하거나(07 상단의 복원 절차 필요), [08 — 정리](08-cleanup.md)로 이동해 리소스를 삭제하세요.

---

## 트러블슈팅

| 증상 | 원인 | 해결 방법 |
|------|------|-----------|
| `kubectl patch` 시 `admission webhook "istio-kube-gateway-validating-webhook.azmk8s.io" denied the request: ... spec.minReplicas must be >= 2` 오류 | ConfigMap의 HPA `minReplicas`를 2 미만으로 지정함 — 허용 목록 검증에 의해 거부됨 | ConfigMap의 `minReplicas`를 2 이상으로 수정한 뒤 다시 patch합니다 |
| `istio-gateway-class-defaults` ConfigMap이 보이지 않음 | 클러스터 생성 직후에는 add-on이 ConfigMap을 배포하기까지 최대 5분이 걸릴 수 있음 | 수 분 대기 후 `kubectl get configmap -n aks-istio-system`으로 다시 확인합니다 |
| parametersRef 적용 후에도 HPA 값이 바뀌지 않음 | 반영에 수십 초가 걸리거나, ConfigMap이 Gateway와 다른 네임스페이스에 있음 | 30초–1분 대기 후 재확인하고, ConfigMap이 `workshop` 네임스페이스에 있는지 `kubectl get configmap gw-options -n workshop`으로 확인합니다 |
| 3.2 수행 후에도 `EXTERNAL-IP`가 공인 IP로 남아 있음 | Azure LB 프런트엔드 재구성에 시간이 걸림 (내부 LB `kubernetes-internal` 생성 포함) | 1–2분 대기 후 `kubectl get svc httpbin-gateway-approuting-istio -n workshop`으로 재확인합니다. `kubectl get events -n workshop --sort-by=.lastTimestamp`로 `EnsuringLoadBalancer` 진행 여부를 볼 수 있습니다 |

---

[← 05 — TLS Gateway와 DNS A 레코드](05-tls-gateway-externaldns.md) | 다음: [07 — AFD 카나리 마이그레이션 (옵션)](07-afd-canary-migration.md) 또는 [08 — 정리](08-cleanup.md)
