# 01 — 사전 준비

> 🟢 실행 = 직접 입력·수행 · 👁️ 예시 = 눈으로만(개념/발췌) · 📋 예상 출력 = 비교용(입력 불필요)

예상 소요 시간: ~10분

---

## 1. Cloud Shell 접속

[portal.azure.com](https://portal.azure.com)에 로그인한 뒤 상단 툴바의 Cloud Shell 아이콘(>_)을 클릭합니다. 셸 유형으로 **Bash**를 선택합니다. 처음 접속 시 스토리지 계정 생성 안내가 표시되면 안내에 따라 완료합니다.

현재 활성 구독을 확인합니다.

🟢 **실행**
```bash
az account show --query '{name:name, id:id}' -o table
# 구독이 여러 개면: az account set --subscription "<구독 이름 또는 ID>"
```

---

## 2. az CLI 버전 확인

이 워크샵은 `--enable-app-routing` 및 Gateway API 관련 플래그를 사용합니다. az CLI **2.86.0 이상**이 필요합니다.

🟢 **실행**
```bash
az version --query '"azure-cli"' -o tsv
```

📋 **예상 출력**
```
2.86.0
```

버전이 낮다면 Cloud Shell은 자동 업데이트되지만 반영에 지연이 있을 수 있습니다. Cloud Shell 세션을 닫고 새 세션을 열어 재확인합니다. 그래도 낮다면 다음을 실행합니다.

🟢 **실행**
```bash
az upgrade
```

---

## 3. 환경변수 정의 및 저장

워크샵 전반에서 사용하는 변수를 정의하고 `~/.approuting-ws-env`에 저장합니다. 이후 모든 모듈의 **0단계**에서 `source ~/.approuting-ws-env`로 변수를 복원합니다.

🟢 **실행**
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

**변수 설명:**

| 변수 | 값 예시 | 설명 |
|------|---------|------|
| `SUFFIX` | `04271` | 리소스 이름 충돌을 방지하는 5자리 난수 |
| `RESOURCE_GROUP` | `rg-approuting-ws-04271` | 워크샵 전용 리소스 그룹 |
| `CLUSTER` | `aks-approuting-04271` | AKS 클러스터 이름 |
| `LOCATION` | `koreacentral` | 배포 지역 |
| `ZONE_NAME` | `ws04271.approuting-workshop.example` | 가상 도메인 (위임 불필요) |
| `KV_NAME` | `kv-apr-04271` | Key Vault 이름 (최대 12자, 전역 고유) |
| `UAMI_NAME` | `id-approuting-04271` | User-assigned Managed Identity 이름 |
| `SA_NAME` | `approuting-ws-sa` | Kubernetes ServiceAccount 이름 |
| `APP_NAMESPACE` | `workshop` | 앱 배포 네임스페이스 |
| `CERT_NAME` | `approuting-ws-cert` | Key Vault 인증서 이름 |

> **참고** `KV_NAME=kv-apr-$SUFFIX`는 접두사 7자 + 5자리 SUFFIX = 최대 12자로, Key Vault 이름 24자 제한 및 전역 고유 요건을 만족합니다.
>
> **참고** `ZONE_NAME`은 실제 DNS에 등록하지 않는 **가상 도메인**입니다. 검증은 Azure Private DNS Zone의 네임서버 직접 질의 방식을 사용하므로 외부 도메인 위임이 필요 없습니다.

---

## 트러블슈팅

| 증상 | 원인 | 해결 방법 |
|------|------|-----------|
| `az version` 결과가 2.86.0 미만 | Cloud Shell 업데이트 반영 지연 | Cloud Shell 세션을 닫고 새 세션을 열거나 `az upgrade`를 실행합니다 |
| Cloud Shell 시작 시 스토리지 계정 오류 | Cloud Shell 전용 스토리지 미구성 | 안내에 따라 스토리지 계정을 새로 생성하거나, 기존 계정을 선택합니다 |
| `az account show` 시 권한 오류 또는 빈 결과 | 구독 접근 권한 부족 | 구독에 **Owner** 또는 **RBAC Administrator + Managed Identity Contributor** 역할이 있는지 확인합니다 (04 모듈에서 역할 할당 및 Federated Identity Credential 생성에 필요) |

---

[← README](../README.md) | 다음: [02 — 환경 준비](02-environment-setup.md)

<!-- TODO(rehearsal): 예상 출력 실측 검증 -->
