# 08 — 정리

> 🟢 실행 = 직접 입력·수행 · 👁️ 예시 = 눈으로만(개념/발췌) · 📋 예상 출력 = 비교용(입력 불필요)

예상 소요 시간: 5–10분 (RG 삭제 완료 대기 포함)

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

이 워크샵에서 직접 생성한 모든 리소스는 단일 리소스 그룹 `$RESOURCE_GROUP`에 속해 있습니다.
Azure DNS Zone과 05 모듈에서 등록한 A 레코드(수동 또는 ExternalDNS 옵션의 자동 발행분)도 동일 RG에 있으므로, RG를 삭제하면 Zone과 함께 레코드가 함께 제거됩니다.
AKS가 관리하는 노드 리소스 그룹(`MC_...`)과 그 안의 리소스(03 모듈 8절의 정적 공인 IP 포함)는 클러스터 삭제 시 자동으로 연쇄 삭제됩니다.

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
az keyvault purge --name $KV_NAME --no-wait
az keyvault list-deleted --query "[?name=='$KV_NAME'].name" -o tsv
```

purge가 접수되면 두 번째 명령의 출력이 비어 있거나, 잠시 후 재실행 시 비게 됩니다.

---

## 4. 로컬 흔적 정리

purge까지 접수된 것을 확인한 뒤 Cloud Shell 환경 파일과 kubectl 컨텍스트를 정리합니다.
(`~/.approuting-ws-env`를 먼저 지우면 purge에 필요한 `$KV_NAME`을 잃게 되므로 반드시 마지막에 수행합니다.)

🟢 **실행**
```bash
kubectl config delete-context $CLUSTER 2>/dev/null || true
rm -f ~/.approuting-ws-env cert-policy.json
```

---

## 5. 과금 종료 확인

리소스 그룹 삭제가 완료된 시점(2단계)부터 과금이 중단됩니다. 단, Azure 포털의 비용 분석 화면에는 수 시간의 반영 지연이 있을 수 있습니다.

---

## 트러블슈팅

| 증상 | 원인 | 해결 방법 |
|------|------|-----------|
| RG 삭제가 10분 이상 걸림 | 정상 — AKS 클러스터의 노드 리소스 그룹(`MC_...`)이 연쇄 삭제되므로 시간이 소요됩니다 | `az group show --name $RESOURCE_GROUP --query properties.provisioningState -o tsv`로 `Deleting` 상태를 확인하며 대기합니다 |
| `az keyvault purge` 실행 시 권한 오류 발생 | Key Vault Purger 역할이 없거나 RG 삭제가 아직 완료되지 않음 | 구독 Owner 또는 Key Vault Purger 역할을 보유한 계정으로 실행합니다. RG 삭제 완료 후 다시 시도하세요 |
| 포털에서 `MC_` 노드 RG가 남아 있는 것처럼 보임 | AKS가 노드 RG를 비동기로 삭제 중이며, 포털 캐시가 아직 갱신되지 않은 것입니다 | 수 분 후 포털을 새로 고침하면 자동으로 사라집니다 |

---

[← 07 — AFD 카나리 마이그레이션 (옵션)](07-afd-canary-migration.md) · [← 06 — Gateway 인프라 커스터마이징 (옵션)](06-gateway-customizations.md) · [← 05 — TLS Gateway와 DNS A 레코드](05-tls-gateway-externaldns.md) | [처음으로 (README)](../README.md)
