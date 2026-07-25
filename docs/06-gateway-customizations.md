# 06 — Gateway 인프라 커스터마이징 (옵션)

> 🟢 실행 = 직접 입력·수행 · 👁️ 예시 = 눈으로만(개념/발췌) · 📋 예상 출력 = 비교용(입력 불필요)

예상 소요 시간: 10–15분

> **옵션 모듈**: 이 모듈은 선택 사항입니다. 건너뛰고 바로 [07 — 정리](07-cleanup.md)로 이동해도 됩니다.

Gateway 리소스를 만들면 application routing add-on(관리형 Istio)이 뒤에서 LoadBalancer **Service**, **Deployment**, **HPA**(HorizontalPodAutoscaler), **PDB**(PodDisruptionBudget)를 자동으로 생성합니다. 03 모듈에서 이미 `httpbin-gateway-approuting-istio`라는 이름으로 이 리소스들을 확인했습니다.

이 모듈에서는 자동 생성된 인프라 리소스를 커스터마이징하는 두 가지 방법을 실습합니다.

1. **Annotation customizations** — `spec.infrastructure.annotations`로 생성된 Service에 어노테이션 전달 (개념 확인)
2. **ConfigMap customizations** — GatewayClass 기본값 확인 및 Gateway별 ConfigMap으로 HPA·Deployment 재정의 (실습 후 원복)

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

## 1. Annotation customizations — 개념 확인

Gateway의 `spec.infrastructure.annotations`에 지정한 어노테이션은 자동 생성되는 LoadBalancer Service에 그대로 전달됩니다. 대표적인 활용 예는 **내부(Internal) Load Balancer** 전환입니다.

👁️ **예시** — 내부 LB로 노출하는 Gateway (적용하지 않습니다)
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
      service.beta.kubernetes.io/azure-load-balancer-internal-subnet: "internal-lb-subnet"
  listeners:
    - name: http
      protocol: HTTP
      port: 80
```

> ⚠️ **이 워크샵의 `httpbin-gateway`에는 적용하지 마세요.** 내부 LB 어노테이션을 추가하면 03 모듈에서 고정한 정적 공인 IP 기반의 외부 접근이 끊어집니다. 여기서는 개념만 확인하고, 다음 절부터는 외부 접근에 영향이 없는 ConfigMap 방식을 실습합니다.

---

## 2. GatewayClass 수준 기본값 확인

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

## 3. Gateway별 커스터마이징 — parametersRef

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

---

## 4. 원복

커스터마이징을 제거하면 GatewayClass 기본값(2/5)으로 되돌아갑니다.

🟢 **실행**
```bash
# 1) Gateway에서 parametersRef 제거
kubectl patch gateway httpbin-gateway -n workshop --type json \
  -p '[{"op":"remove","path":"/spec/infrastructure/parametersRef"}]'

# 2) 커스터마이징 ConfigMap 삭제
kubectl delete configmap gw-options -n workshop
```

📋 **예상 출력**
```
gateway.gateway.networking.k8s.io/httpbin-gateway patched
configmap "gw-options" deleted
```

약 30초 후 확인합니다.

🟢 **실행**
```bash
kubectl get hpa httpbin-gateway-approuting-istio -n workshop
```

📋 **예상 출력**
```
NAME                               REFERENCE                                     TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
httpbin-gateway-approuting-istio   Deployment/httpbin-gateway-approuting-istio   cpu: 2%/80%   2         5         2          41m
```

---

## 트러블슈팅

| 증상 | 원인 | 해결 방법 |
|------|------|-----------|
| `kubectl patch` 시 `admission webhook "istio-kube-gateway-validating-webhook.azmk8s.io" denied the request: ... spec.minReplicas must be >= 2` 오류 | ConfigMap의 HPA `minReplicas`를 2 미만으로 지정함 — 허용 목록 검증에 의해 거부됨 | ConfigMap의 `minReplicas`를 2 이상으로 수정한 뒤 다시 patch합니다 |
| `istio-gateway-class-defaults` ConfigMap이 보이지 않음 | 클러스터 생성 직후에는 add-on이 ConfigMap을 배포하기까지 최대 5분이 걸릴 수 있음 | 수 분 대기 후 `kubectl get configmap -n aks-istio-system`으로 다시 확인합니다 |
| parametersRef 적용 후에도 HPA 값이 바뀌지 않음 | 반영에 수십 초가 걸리거나, ConfigMap이 Gateway와 다른 네임스페이스에 있음 | 30초–1분 대기 후 재확인하고, ConfigMap이 `workshop` 네임스페이스에 있는지 `kubectl get configmap gw-options -n workshop`으로 확인합니다 |

---

[← 05 — TLS Gateway와 DNS A 레코드](05-tls-gateway-externaldns.md) | 다음: [07 — 정리](07-cleanup.md)
