# 05 — TLS Gateway와 ClusterExternalDNS

> 🟢 실행 = 직접 입력·수행 · 👁️ 예시 = 눈으로만(개념/발췌) · 📋 예상 출력 = 비교용(입력 불필요)

예상 소요 시간: 15–20분 (인증서 동기화·A 레코드 생성 대기 포함)

---

<details>
<summary>🔄 0단계 — 변수 재설정 (새 터미널/세션에서 시작하는 경우)</summary>

🟢 **실행**
```bash
source ~/.approuting-ws-env
echo "CERT_URI=$CERT_URI"
```

🟢 **실행**
```bash
az aks get-credentials --resource-group $RESOURCE_GROUP --name $CLUSTER --overwrite-existing || true
```

</details>

---

## 1. TLS Gateway 적용

이 단계에서는 03 모듈에서 생성한 `httpbin-gateway`를 HTTPS 리스너가 추가된 버전으로 덮어씁니다.
`manifests/gateway-tls.yaml`에는 두 개의 `tls.options` 키가 포함되어 있습니다.

👁️ **예시**
```yaml
tls:
  mode: Terminate
  options:
    kubernetes.azure.com/tls-cert-keyvault-uri: ${CERT_URI}
    kubernetes.azure.com/tls-cert-service-account: ${SA_NAME}
```

- **`tls-cert-keyvault-uri`**: Application Routing operator가 Key Vault에서 인증서를 가져올 버전 없는(versionless) URI입니다. operator가 이 URI를 감지하면 `SecretProviderClass`를 자동 생성합니다.
- **`tls-cert-service-account`**: 인증서를 읽을 때 사용할 Kubernetes ServiceAccount를 지정합니다. 이 SA는 04 모듈에서 생성하고 UAMI와 Federated Identity Credential로 연결한 SA(`$SA_NAME`)입니다.

`envsubst`로 변수(`$CERT_URI`, `$SA_NAME`, `$ZONE_NAME`)를 치환한 뒤 적용합니다.

🟢 **실행**
```bash
cd ~/ms-aks-approuting-workshop01
envsubst < manifests/gateway-tls.yaml | kubectl apply -n $APP_NAMESPACE -f -
kubectl wait -n $APP_NAMESPACE --for=condition=programmed gateway httpbin-gateway --timeout=300s
```

---

## 2. Reconcile 결과 관찰

Gateway에 `tls-cert-keyvault-uri`가 적용되면 Application Routing operator가 자동으로 `SecretProviderClass`를 생성하고, Secrets Store CSI Driver가 Key Vault에서 인증서를 동기화해 `kubernetes.io/tls` Secret을 생성합니다.

🟢 **실행**
```bash
kubectl get secretproviderclass,secret -n $APP_NAMESPACE
kubectl get gateway httpbin-gateway -n $APP_NAMESPACE -o jsonpath='{.spec.listeners[?(@.name=="https")].tls.certificateRefs}' && echo
```

📋 **예상 출력**
```
NAME                                                                              AGE
secretproviderclass.secrets-store.csi.x-k8s.io/kv-gw-cert-httpbin-gateway-https   1m

NAME                                      TYPE                DATA   AGE
secret/kv-gw-cert-httpbin-gateway-https   kubernetes.io/tls   2      1m

[{"group":"","kind":"Secret","name":"kv-gw-cert-httpbin-gateway-https","namespace":"workshop"}]
```

`SecretProviderClass`와 동일한 이름(`kv-gw-cert-httpbin-gateway-https`)의 `kubernetes.io/tls` Secret이 생성되고,
Gateway의 `certificateRefs`가 이 Secret을 참조하면 HTTPS 리스너가 활성화됩니다.
operator는 인증서 동기화를 담당하는 같은 이름의 Deployment(`kv-gw-cert-httpbin-gateway-https`)도 함께 생성합니다.

---

## 3. ClusterExternalDNS 적용

> **옵션** 03 모듈 8절에서 Gateway에 **정적 공인 IP**를 고정했다면, 3–4단계 대신 [6절 옵션 경로](#6-옵션--정적-ip--수동-a-레코드-externaldns-생략)로 A 레코드를 수동 등록해도 됩니다. IP가 바뀌지 않으므로 DNS 자동화 없이 레코드 1건이면 충분합니다.

`ClusterExternalDNS`는 Application Routing 애드온이 관리하는 ExternalDNS 인스턴스를 클러스터 수준에서 선언하는 리소스입니다.

👁️ **예시**
```yaml
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

- **`resourceNamespace`**: ExternalDNS 파드가 배포될 네임스페이스입니다. operator는 이 네임스페이스에 managed ExternalDNS Deployment를 생성하므로, SA(`$SA_NAME`)도 동일한 네임스페이스에 있어야 합니다.
- **`resourceTypes: [gateway]`**: HTTPRoute의 호스트명을 Gateway를 기준으로 감시해 DNS A 레코드를 생성합니다.
- **`identity.serviceAccount`**: DNS Zone Contributor 역할을 가진 UAMI와 FIC로 연결된 SA를 지정합니다.

🟢 **실행**
```bash
envsubst < manifests/cluster-external-dns.yaml | kubectl apply -f -
kubectl get pods -n $APP_NAMESPACE -l app=workshop-cluster-dns-external-dns 2>/dev/null; kubectl get clusterexternaldns
```

📋 **예상 출력**
```
NAME                                                  READY   STATUS    RESTARTS   AGE
workshop-cluster-dns-external-dns-7f7d58bfc8-4cq9h    1/1     Running   0          44s

NAME                   AGE
workshop-cluster-dns   34s
```

> **참고** operator는 `resourceName` 뒤에 `-external-dns`를 붙인 이름(`workshop-cluster-dns-external-dns`)으로 Deployment와 파드를 생성합니다.

---

## 4. A 레코드 확인

ExternalDNS 파드가 Gateway와 HTTPRoute의 호스트명을 감지해 Azure DNS에 A 레코드를 등록하기까지 약 1분 정도 걸릴 수 있습니다.

🟢 **실행**
```bash
az network dns record-set a list --resource-group $RESOURCE_GROUP --zone-name $ZONE_NAME -o table
```

📋 **예상 출력**
```
TTL    Fqdn                                          Name     ProvisioningState    ResourceGroup
-----  --------------------------------------------  -------  -------------------  ----------------------
300    httpbin.ws20692.approuting-workshop.example.  httpbin  Succeeded            rg-approuting-ws-20692
```

`httpbin` A 레코드 1건이 TTL 300으로 등록되어 있으면 정상입니다.

---

## 5. End-to-End HTTPS 검증

> ⏳ 이 단계를 진행하기 전에 4단계 A 레코드가 확인되었는지 먼저 확인합니다.

🟢 **실행**
```bash
NS=$(az network dns zone show --resource-group $RESOURCE_GROUP --name $ZONE_NAME --query 'nameServers[0]' -o tsv | sed 's/\.$//')
GATEWAY_IP=$(dig +short @${NS} httpbin.${ZONE_NAME} | tail -1)
echo "NS=$NS / IP=$GATEWAY_IP"
curl -k -I --resolve "httpbin.${ZONE_NAME}:443:${GATEWAY_IP}" "https://httpbin.${ZONE_NAME}/get"
```

📋 **예상 출력**
```
NS=ns1-04.azure-dns.com / IP=20.249.51.10
HTTP/2 200 
access-control-allow-credentials: true
access-control-allow-origin: *
content-type: application/json; charset=utf-8
date: Sat, 25 Jul 2026 07:22:08 GMT
x-envoy-upstream-service-time: 0
server: istio-envoy
```

> **📚 교육 포인트**
>
> - **`dig @NS` — 권한 네임서버에 직접 질의하는 이유**: 이 워크샵은 위임(NS 등록) 없는 가상 도메인을 사용합니다. 인터넷의 일반 재귀 리졸버는 위임이 없으면 레코드를 찾을 수 없습니다. Azure DNS 존의 권한 네임서버에 직접 질의(`@NS`)하면 위임 없이도 A 레코드를 조회할 수 있습니다. 실 운영에서는 등록기관에서 NS를 위임하면 일반 `dig httpbin.<zone>`으로도 확인 가능합니다.
>
> - **`-k` 플래그가 필요한 이유**: 04 모듈에서 생성한 인증서는 Key Vault 자체 서명(Self-signed) 인증서입니다. 신뢰할 수 있는 CA가 서명하지 않았으므로 curl의 기본 인증서 체인 검증이 실패합니다. `-k`(또는 `--insecure`)는 이 검증을 건너뜁니다. CA 서명 인증서를 사용하는 실 운영 환경에서는 `-k` 대신 `--cacert <ca.crt>`로 루트 CA를 명시해 체인을 검증합니다.
>
> - **`--resolve` — 로컬 DNS를 우회하는 테스트 기법**: `--resolve <host>:<port>:<ip>` 옵션은 DNS 조회 없이 특정 호스트를 지정한 IP로 강제 연결합니다. 로컬 리졸버에 레코드가 전파되기 전이나 위임이 없는 환경에서 실제 TLS 핸드셰이크와 HTTP 응답까지 검증하는 데 유용합니다.

---

## 6. 옵션 — 정적 IP + 수동 A 레코드 (ExternalDNS 생략)

> **전제** 03 모듈 8절(정적 공인 IP 옵션)을 수행한 경우에만 해당합니다. 이 경로는 **3–4단계를 대체**합니다. 1–2단계(TLS Gateway·인증서 동기화)는 IP와 무관하므로 그대로 수행해야 합니다.

Gateway IP가 고정이면 DNS 레코드가 바뀔 일이 없으므로, ExternalDNS 없이 A 레코드를 한 번만 수동 등록하면 됩니다.
`ClusterExternalDNS` 리소스·managed ExternalDNS 파드·레코드 생성 대기가 모두 불필요해지고, 클러스터를 재생성해도(같은 IP를 다시 고정하는 한) 레코드를 건드릴 필요가 없습니다.

🟢 **실행**
```bash
export NODE_RG=$(az aks show --resource-group $RESOURCE_GROUP --name $CLUSTER --query nodeResourceGroup -o tsv)
export STATIC_IP=$(az network public-ip show --resource-group $NODE_RG --name pip-httpbin-gateway --query ipAddress -o tsv)
az network dns record-set a add-record \
  --resource-group $RESOURCE_GROUP \
  --zone-name $ZONE_NAME \
  --record-set-name httpbin \
  --ipv4-address $STATIC_IP \
  --ttl 300 --query fqdn -o tsv
az network dns record-set a list --resource-group $RESOURCE_GROUP --zone-name $ZONE_NAME -o table
```

📋 **예상 출력**
```
httpbin.ws15441.approuting-workshop.example.
TTL    Fqdn                                          Name     ProvisioningState    ResourceGroup
-----  --------------------------------------------  -------  -------------------  ----------------------
300    httpbin.ws15441.approuting-workshop.example.  httpbin  Succeeded            rg-approuting-ws-15441
```

레코드가 등록되면 [5단계 End-to-End HTTPS 검증](#5-end-to-end-https-검증)을 동일하게 수행합니다. 06 모듈의 정리 절차도 변경 없이 그대로 적용됩니다(레코드는 RG 삭제 시 함께 제거).

> **📚 교육 포인트 — 두 패턴 비교**
>
> | | ClusterExternalDNS (3–4단계) | 정적 IP + 수동 A 레코드 (이 옵션) |
> |---|---|---|
> | DNS 갱신 | Gateway IP 변경을 자동 추적 | 불필요 (IP가 고정) |
> | 구성 요소 | ClusterExternalDNS + managed 파드 | 정적 공인 IP 1개 |
> | 호스트 추가 시 | HTTPRoute만 추가하면 자동 등록 | 레코드를 수동으로 추가 |
> | 어울리는 환경 | 호스트가 많거나 IP가 유동적인 클러스터 | 방화벽 허용 목록 등 IP 고정이 전제인 환경 |
>
> ExternalDNS는 자동화·다중 호스트에, 정적 IP는 프런트엔드 주소가 계약된 운영 환경에 적합합니다. 두 방식은 함께 써도 됩니다 — ExternalDNS는 정적 IP도 그대로 레코드에 반영합니다.

---

## 트러블슈팅

| 증상 | 원인 | 해결 방법 |
|------|------|-----------|
| Secret은 물론 `SecretProviderClass`도 전혀 생성되지 않고 `kubectl get clusterexternaldns`가 CRD 없음 오류를 반환 | application routing 애드온(operator)이 비활성화됨 — `--enable-app-routing-istio`만으로는 operator가 켜지지 않음 | `az aks show -g $RESOURCE_GROUP -n $CLUSTER --query ingressProfile.webAppRouting.enabled -o tsv`가 `false`이면 `az aks approuting enable -g $RESOURCE_GROUP -n $CLUSTER`를 실행합니다(2–10분 소요). 완료 후 operator가 기존 Gateway 어노테이션을 자동으로 reconcile합니다 |
| `kubectl get secret -n $APP_NAMESPACE`에 `kv-gw-cert-*` Secret이 생성되지 않음 | SA의 `azure.workload.identity/client-id` annotation 누락, `azure.workload.identity/use: "true"` label 누락, 또는 FIC `subject` 불일치 | `kubectl describe secretproviderclass -n $APP_NAMESPACE`로 오류 메시지를 확인합니다. `kubectl get sa $SA_NAME -n $APP_NAMESPACE -o yaml`로 annotation/label을 점검하고, `az identity federated-credential list --identity-name $UAMI_NAME -g $RESOURCE_GROUP`으로 `subject` 값이 `system:serviceaccount:$APP_NAMESPACE:$SA_NAME`과 일치하는지 확인합니다 |
| A 레코드가 생성되지 않음 | ExternalDNS 파드의 DNS Zone Contributor 권한이 아직 전파 중이거나, `ClusterExternalDNS`의 `dnsZoneResourceIDs`가 올바르지 않음 | `kubectl logs -n $APP_NAMESPACE deploy/workshop-cluster-dns-external-dns`로 ExternalDNS 로그를 확인합니다. `az role assignment list --scope $ZONE_ID`로 UAMI에 DNS Zone Contributor가 부여되었는지 확인합니다. RBAC 전파에 최대 5분이 소요될 수 있습니다 |
| `curl: (60) SSL certificate problem` 오류 발생 | `-k` 옵션 없이 자체 서명 인증서 엔드포인트에 접속함 | 검증 명령에 `-k` 플래그를 추가합니다. 실 운영에서 CA 서명 인증서를 사용한다면 `-k` 없이 `--cacert <ca.crt>`로 올바른 루트 CA를 지정하세요 |
| `envsubst` 적용 후 YAML에 빈 값(`${CERT_URI}` 등이 그대로 남음)이 나타남 | 0단계를 수행하지 않아 환경 변수가 설정되지 않음 | 0단계의 `source ~/.approuting-ws-env`를 실행한 뒤 `echo $CERT_URI`로 값이 채워졌는지 확인합니다 |

---

[← 04 — DNS·TLS 인프라 준비](04-dns-tls-infra.md) | 다음: [06 — 정리](06-cleanup.md)
