# 06 — 정리

> 🟢 실행 = 직접 입력·수행 · 👁️ 예시 = 눈으로만(개념/발췌) · 📋 예상 출력 = 비교용(입력 불필요)

예상 소요 시간: ~5분 (삭제는 백그라운드 진행)

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

이 워크샵에서 생성한 모든 리소스는 단일 리소스 그룹 `$RESOURCE_GROUP`에 속해 있습니다.
Azure DNS Zone과 ClusterExternalDNS가 생성한 A 레코드도 동일 RG에 있으므로, RG를 삭제하면 Zone과 함께 레코드가 함께 제거됩니다.

> **참고**: ClusterExternalDNS 리소스 자체를 삭제해도 이미 생성된 DNS 레코드는 자동으로 삭제되지 않습니다. ExternalDNS는 자신이 관리하지 않는다고 판단한 레코드를 보존하는 보수적인 정책을 사용하기 때문입니다. RG 수준의 삭제가 레코드까지 완전히 제거하는 가장 확실한 방법입니다.

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

## 2. Key Vault soft-delete 정리

Azure Key Vault는 기본적으로 soft-delete가 활성화되어 있습니다. RG를 삭제해도 Key Vault는 90일간 삭제된(soft-deleted) 상태로 보존되며, 동일한 이름으로 새 Key Vault를 생성하려면 purge(완전 삭제)가 필요합니다.

RG 삭제가 완전히 완료된 뒤 다음 명령으로 purge를 수행합니다. 삭제가 아직 진행 중이라면 오류 메시지가 출력되며, 잠시 후 다시 시도하면 됩니다.

🟢 **실행**
```bash
az keyvault purge --name $KV_NAME --no-wait 2>/dev/null || echo "삭제 완료 후 다시 시도하세요"
```

---

## 3. 로컬 흔적 정리

Cloud Shell 환경 파일과 kubectl 컨텍스트를 정리합니다.

🟢 **실행**
```bash
rm -f ~/.approuting-ws-env cert-policy.json
kubectl config delete-context $CLUSTER 2>/dev/null || true
```

---

## 4. 과금 종료 확인

리소스 그룹 삭제가 완료되면 과금이 중단됩니다. 단, Azure 포털의 비용 분석 화면에는 수 시간의 반영 지연이 있을 수 있습니다.

RG 삭제 완료 여부는 다음 명령으로 확인합니다. `false`가 출력되면 삭제가 완료된 것입니다.

🟢 **실행**
```bash
az group exists --name $RESOURCE_GROUP
```

📋 **예상 출력**
```
false
```

---

## 트러블슈팅

| 증상 | 원인 | 해결 방법 |
|------|------|-----------|
| RG 삭제가 10분 이상 걸림 | 정상 — AKS 클러스터의 노드 리소스 그룹(`MC_...`)이 연쇄 삭제되므로 시간이 소요됩니다 | `az group show --name $RESOURCE_GROUP --query properties.provisioningState -o tsv`로 `Deleting` 상태를 확인하며 대기합니다 |
| `az keyvault purge` 실행 시 권한 오류 발생 | Key Vault Purger 역할이 없거나 RG 삭제가 아직 완료되지 않음 | 구독 Owner 또는 Key Vault Purger 역할을 보유한 계정으로 실행합니다. RG 삭제 완료 후 다시 시도하세요 |
| 포털에서 `MC_` 노드 RG가 남아 있는 것처럼 보임 | AKS가 노드 RG를 비동기로 삭제 중이며, 포털 캐시가 아직 갱신되지 않은 것입니다 | 수 분 후 포털을 새로 고침하면 자동으로 사라집니다 |

---

[← 05 — TLS Gateway와 ClusterExternalDNS](05-tls-gateway-externaldns.md) | [처음으로 (README)](../README.md)

<!-- TODO(rehearsal): 예상 출력 실측 검증 -->
