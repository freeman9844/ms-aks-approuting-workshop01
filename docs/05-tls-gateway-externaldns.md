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
NAME                                                       AGE
secretproviderclass.secrets-store.csi.x-k8s.io/kv-gw-cert-httpbin-gateway-https   ...

NAME                                           TYPE                DATA   AGE
secret/kv-gw-cert-httpbin-gateway-https       kubernetes.io/tls   2      ...

[{"name":"kv-gw-cert-httpbin-gateway-https","namespace":"workshop"}]
```

`SecretProviderClass`와 동일한 이름(`kv-gw-cert-httpbin-gateway-https`)의 `kubernetes.io/tls` Secret이 생성되고,
Gateway의 `certificateRefs`가 이 Secret을 참조하면 HTTPS 리스너가 활성화됩니다.

---

## 3. ClusterExternalDNS 적용

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
kubectl get pods -n $APP_NAMESPACE -l app=workshop-cluster-dns 2>/dev/null; kubectl get clusterexternaldns
```

---

## 4. A 레코드 확인

ExternalDNS 파드가 Gateway와 HTTPRoute의 호스트명을 감지해 Azure DNS에 A 레코드를 등록하기까지 약 1분 정도 걸릴 수 있습니다.

🟢 **실행**

```bash
az network dns record-set a list --resource-group $RESOURCE_GROUP --zone-name $ZONE_NAME -o table
```

📋 **예상 출력**

```
Name      ResourceGroup    Ttl    Type    ProvisioningState    Fqdn
--------  ---------------  -----  ------  -------------------  ----------------------
httpbin   <rg>             300    A       Succeeded            httpbin.<zone>.
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
NS=ns1-xx.azure-dns.com / IP=<공인 IP>
HTTP/2 200
server: istio-envoy
...
```

> **📚 교육 포인트**
>
> - **`dig @NS` — 권한 네임서버에 직접 질의하는 이유**: 이 워크샵은 위임(NS 등록) 없는 가상 도메인을 사용합니다. 인터넷의 일반 재귀 리졸버는 위임이 없으면 레코드를 찾을 수 없습니다. Azure DNS 존의 권한 네임서버에 직접 질의(`@NS`)하면 위임 없이도 A 레코드를 조회할 수 있습니다. 실 운영에서는 등록기관에서 NS를 위임하면 일반 `dig httpbin.<zone>`으로도 확인 가능합니다.
>
> - **`-k` 플래그가 필요한 이유**: 04 모듈에서 생성한 인증서는 Key Vault 자체 서명(Self-signed) 인증서입니다. 신뢰할 수 있는 CA가 서명하지 않았으므로 curl의 기본 인증서 체인 검증이 실패합니다. `-k`(또는 `--insecure`)는 이 검증을 건너뜁니다. CA 서명 인증서를 사용하는 실 운영 환경에서는 `-k` 대신 `--cacert <ca.crt>`로 루트 CA를 명시해 체인을 검증합니다.
>
> - **`--resolve` — 로컬 DNS를 우회하는 테스트 기법**: `--resolve <host>:<port>:<ip>` 옵션은 DNS 조회 없이 특정 호스트를 지정한 IP로 강제 연결합니다. 로컬 리졸버에 레코드가 전파되기 전이나 위임이 없는 환경에서 실제 TLS 핸드셰이크와 HTTP 응답까지 검증하는 데 유용합니다.

---

## 트러블슈팅

| 증상 | 원인 | 해결 방법 |
|------|------|-----------|
| `kubectl get secret -n $APP_NAMESPACE`에 `kv-gw-cert-*` Secret이 생성되지 않음 | SA의 `azure.workload.identity/client-id` annotation 누락, `azure.workload.identity/use: "true"` label 누락, 또는 FIC `subject` 불일치 | `kubectl describe secretproviderclass -n $APP_NAMESPACE`로 오류 메시지를 확인합니다. `kubectl get sa $SA_NAME -n $APP_NAMESPACE -o yaml`로 annotation/label을 점검하고, `az identity federated-credential list --identity-name $UAMI_NAME -g $RESOURCE_GROUP`으로 `subject` 값이 `system:serviceaccount:$APP_NAMESPACE:$SA_NAME`과 일치하는지 확인합니다 |
| A 레코드가 생성되지 않음 | ExternalDNS 파드의 DNS Zone Contributor 권한이 아직 전파 중이거나, `ClusterExternalDNS`의 `dnsZoneResourceIDs`가 올바르지 않음 | `kubectl logs -n $APP_NAMESPACE deploy/workshop-cluster-dns`로 ExternalDNS 로그를 확인합니다. `az role assignment list --scope $ZONE_ID`로 UAMI에 DNS Zone Contributor가 부여되었는지 확인합니다. RBAC 전파에 최대 5분이 소요될 수 있습니다 |
| `curl: (60) SSL certificate problem` 오류 발생 | `-k` 옵션 없이 자체 서명 인증서 엔드포인트에 접속함 | 검증 명령에 `-k` 플래그를 추가합니다. 실 운영에서 CA 서명 인증서를 사용한다면 `-k` 없이 `--cacert <ca.crt>`로 올바른 루트 CA를 지정하세요 |
| `envsubst` 적용 후 YAML에 빈 값(`${CERT_URI}` 등이 그대로 남음)이 나타남 | 0단계를 수행하지 않아 환경 변수가 설정되지 않음 | 0단계의 `source ~/.approuting-ws-env`를 실행한 뒤 `echo $CERT_URI`로 값이 채워졌는지 확인합니다 |

---

[← 04 — DNS·TLS 인프라 준비](04-dns-tls-infra.md) | 다음: [06 — 다음 모듈](06-next.md)

<!-- TODO(rehearsal): 예상 출력 실측 검증 -->
