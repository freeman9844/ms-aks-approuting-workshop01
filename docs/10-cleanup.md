# 10 — 정리

> 🟢 실행 = 직접 입력·수행 · 👁️ 예시 = 눈으로만(개념/발췌) · 📋 예상 출력 = 비교용(입력 불필요)

예상 소요 시간: 5–10분 (RG 삭제 완료 대기 포함)

09 옵션을 수행했다면 기존 Application Routing AKS와 별도로 `$ISTIO_CLUSTER`가 같은 `$RESOURCE_GROUP`에 남아 있습니다. 두 번째 클러스터의 Istio control plane, `Gateway/istio-session-gateway`, 테스트 앱, Standard Load Balancer와 두 AKS의 노드 리소스 그룹은 아래 RG 삭제 과정에서 함께 제거됩니다.

---

<details>
<summary>🔄 0단계 — 변수 재설정 (새 터미널/세션에서 시작하는 경우)</summary>

🟢 **실행**
```bash
source ~/.approuting-ws-env
```

</details>

---

## 1. 리소스 그룹 삭제

[08 경로](08-private-link-service.md)에서는 소비자 VNet, Private Endpoint, ACI가 `$RESOURCE_GROUP`에 생성됩니다. 반면 PLS와 내부 LB는 원래 Application Routing AKS의 노드 리소스 그룹(`MC_...`)에 생성되며, 해당 클러스터를 삭제하면 함께 제거됩니다.

이 워크샵에서 직접 생성한 관리 리소스는 모두 단일 리소스 그룹 `$RESOURCE_GROUP`에 속해 있습니다. 여기에는 원래 Application Routing AKS `$CLUSTER`, 09에서 추가한 `$ISTIO_CLUSTER`, Azure DNS Zone, 05 모듈에서 등록한 A 레코드(수동 또는 ExternalDNS 옵션의 자동 발행분), 그리고 08 경로의 소비자 VNet/Private Endpoint/ACI가 포함됩니다. 따라서 RG를 삭제하면 Zone과 레코드도 함께 제거됩니다.

대신 각 AKS는 자신만의 AKS 생성 노드 리소스 그룹(`MC_<resourceGroup>_<clusterName>_<region>`)을 하나씩 만듭니다. 03 모듈 8절의 정적 공인 IP, 08 경로의 PLS·내부 LB, 09 경로의 Istio용 Standard Load Balancer 같은 인프라는 이 `MC_...` 그룹들 안에 있습니다. `az group delete --name $RESOURCE_GROUP`로 두 managed cluster 리소스를 삭제하면, AKS가 각 클러스터의 `MC_...` 노드 리소스 그룹도 연쇄 삭제합니다.

> **참고**: (05 모듈 5절 옵션을 수행한 경우) ClusterExternalDNS 리소스 자체를 삭제해도 이미 생성된 DNS 레코드는 자동으로 삭제되지 않습니다. ExternalDNS는 자신이 관리하지 않는다고 판단한 레코드를 보존하는 보수적인 정책을 사용하기 때문입니다. RG 수준의 삭제가 레코드까지 완전히 제거하는 가장 확실한 방법입니다.

🟢 **실행**
```bash
az group delete --name $RESOURCE_GROUP --yes --no-wait
az group show --name $RESOURCE_GROUP --query properties.provisioningState -o tsv
```

📋 **예상 출력**
```
Deleting
```

---

## 2. RG 삭제 완료 대기

Key Vault purge는 Vault 삭제가 끝난 뒤에만 가능하므로, RG 삭제 완료를 먼저 확인합니다.
AKS 노드 리소스 그룹 연쇄 삭제로 5–10분가량 걸립니다. `false`가 출력되면 삭제가 완료된 것입니다.

🟢 **실행**
```bash
while [ "$(az group exists --name $RESOURCE_GROUP)" = "true" ]; do echo "삭제 진행 중... 30초 후 재확인"; sleep 30; done
az group exists --name $RESOURCE_GROUP
```

📋 **예상 출력**
```
false
```

> ⏳ 대기하는 동안 이 모듈의 3단계 이후 내용을 미리 읽어두세요.

---

## 3. Key Vault soft-delete 정리

Azure Key Vault는 기본적으로 soft-delete가 활성화되어 있습니다. RG를 삭제해도 Key Vault는 90일간 삭제된(soft-deleted) 상태로 보존되며, 동일한 이름으로 새 Key Vault를 생성하려면 purge(완전 삭제)가 필요합니다.

🟢 **실행**
```bash
if [ -n "${KV_NAME:-}" ] && \
  [ "$(az keyvault list-deleted \
    --query "[?name=='$KV_NAME'] | length(@)" \
    -o tsv)" = "1" ]; then
  az keyvault purge --name "$KV_NAME" --no-wait
  az keyvault list-deleted \
    --query "[?name=='$KV_NAME'].name" \
    -o tsv
else
  echo "purge할 삭제된 Key Vault가 없습니다."
fi
```

purge가 접수되면 두 번째 명령의 출력이 비어 있거나, 잠시 후 재실행 시 비게 됩니다. 04 모듈에서 Key Vault를 만들지 않고 02 → 09 → 10만 수행했다면 `KV_NAME`이 비어 있거나 삭제된 Vault가 없으므로 `purge할 삭제된 Key Vault가 없습니다.`가 정상입니다.

---

## 4. 로컬 흔적 정리

purge까지 접수된 것을 확인한 뒤 Cloud Shell 환경 파일과 kubectl 컨텍스트를 정리합니다.
(`~/.approuting-ws-env`를 먼저 지우면 purge에 필요한 `$KV_NAME`을 잃게 되므로 반드시 마지막에 수행합니다.)

🟢 **실행**
```bash
if [ -n "${ISTIO_CLUSTER:-}" ]; then
  kubectl config delete-context "$ISTIO_CLUSTER" 2>/dev/null || true
fi
kubectl config delete-context "$CLUSTER" 2>/dev/null || true
rm -f ~/.approuting-ws-env ~/cert-policy.json
```

---

## 5. 과금 종료 확인

리소스 그룹 삭제가 완료된 시점(2단계)부터 과금이 중단됩니다. 단, Azure 포털의 비용 분석 화면에는 수 시간의 반영 지연이 있을 수 있습니다.

---

## 트러블슈팅

| 증상 | 원인 | 해결 방법 |
|------|------|-----------|
| RG 삭제가 10분 이상 걸림 | 정상 — 원래 AKS와 09의 추가 AKS가 각각 자신의 노드 리소스 그룹(`MC_...`)을 연쇄 삭제하므로 시간이 더 걸릴 수 있습니다 | `az group show --name $RESOURCE_GROUP --query properties.provisioningState -o tsv`로 `Deleting` 상태를 확인하며 대기합니다 |
| `az keyvault purge` 실행 시 권한 오류 발생 | Key Vault Purger 역할이 없거나 RG 삭제가 아직 완료되지 않음 | 구독 Owner 또는 Key Vault Purger 역할을 보유한 계정으로 실행합니다. RG 삭제 완료 후 다시 시도하세요 |
| `purge할 삭제된 Key Vault가 없습니다.`가 출력됨 | 04 모듈을 건너뛰었거나, 이미 purge가 끝나 삭제된 Key Vault가 더 이상 없음 | 02 → 09 → 10 경로처럼 Key Vault를 만들지 않은 경우 정상입니다. 04를 수행했다면 `az keyvault list-deleted --query "[?name=='$KV_NAME'].name" -o tsv`로 실제 soft-deleted Vault가 있는지 확인합니다 |
| `kubectl config delete-context`가 한쪽 또는 양쪽 모두에서 아무 것도 삭제하지 못함 | 09를 건너뛰어 `$ISTIO_CLUSTER`가 비어 있거나, 이미 컨텍스트를 수동 삭제했거나, 현재 Cloud Shell에 해당 kubeconfig 항목이 없음 | 스크립트의 `2>/dev/null || true` 처리 덕분에 정상적으로 넘어가면 됩니다. 필요하면 `kubectl config get-contexts -o name`으로 남은 컨텍스트만 확인합니다 |
| 포털에서 `MC_` 노드 RG가 남아 있는 것처럼 보임 | AKS가 노드 RG를 비동기로 삭제 중이며, 포털 캐시가 아직 갱신되지 않은 것입니다 | 수 분 후 포털을 새로 고침하면 자동으로 사라집니다 |

---

[← 09 — Istio Gateway API 쿠키 일관 해시·응답 헤더·본문 검증 (옵션)](09-istio-cookie-affinity.md) · [← 08 — Gateway API Private Link Service 검증 (옵션)](08-private-link-service.md) · [← 07 — AFD 카나리 마이그레이션 (옵션)](07-afd-canary-migration.md) · [← 06 — Gateway 인프라 커스터마이징 (옵션)](06-gateway-customizations.md) · [← 05 — TLS Gateway와 DNS A 레코드](05-tls-gateway-externaldns.md) | [처음으로 (README)](../README.md)
