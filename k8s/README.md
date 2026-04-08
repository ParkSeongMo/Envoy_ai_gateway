## Envoy AI Gateway (사내 K8s) 설치/구동

이 폴더는 사내 Kubernetes 환경에서 Envoy AI Gateway를 `envoy-ai-gateway-system` 네임스페이스에 두고, **포트 80**으로 `Gateway`를 띄우기 위한 구성입니다.

### 0) 컨트롤러 설치 여부 확인(이미 설치돼 있으면 스킵)

컨트롤러가 이미 설치돼 있으면(예: `ai-gateway-controller` Pod가 `Running`) 아래 Helm 설치 단계는 건너뛰고, 바로 `k8s/gateway/`만 적용하면 됩니다.

```bash
k get pods -n envoy-ai-gateway-system
```

### 1) Helm으로 CRD + Controller 설치(미설치 시에만)

```bash
helm upgrade -i aieg-crd oci://docker.io/envoyproxy/ai-gateway-crds-helm \
  --version v0.0.0-latest \
  --namespace envoy-ai-gateway-system \
  --create-namespace

helm upgrade -i aieg oci://docker.io/envoyproxy/ai-gateway-helm \
  --version v0.0.0-latest \
  --namespace envoy-ai-gateway-system \
  --create-namespace \
  -f k8s/values.yaml

k wait --timeout=2m -n envoy-ai-gateway-system deployment/ai-gateway-controller --for=condition=Available
```

### 2) 게이트웨이 리소스 적용 (포트 80)

`Gateway` 이름은 `psm-llm-gateway`, 라우트 CR은 `psm-llm-gateway-routes` 입니다.

```bash
k apply -k k8s/gateway
k wait pods --timeout=2m \
  -l gateway.envoyproxy.io/owning-gateway-name=psm-llm-gateway \
  -n envoy-gateway-system \
  --for=condition=Ready
```

추가로 `/test/api1` 경로는 `26-httproute-test-api1.yaml`의 `HTTPRoute`로 `uplus-mready-model` 네임스페이스의 `fastapi-greet:8000`으로 라우팅됩니다.

`/gpt-oss-120b/v1/...` 처럼 경로에 모델 프리픽스가 붙어 들어오면, `27-httproute-gpt-oss-120b.yaml`에서 **`URLRewrite`로 `/gpt-oss-120b`를 제거**한 뒤 `gpt-oss-120b:80` 서비스로 보냅니다(헤더 `mlp-model-name: gpt-oss-120b` 필요).

예전에 `HTTPRoute` 이름 `psm-llm-gateway-gpt-oss-120b-strip` 로 적용한 적이 있으면 `k delete httproute psm-llm-gateway-gpt-oss-120b-strip -n envoy-ai-gateway-system` 으로 정리하세요.

인프라에서 이미 `/v1/chat/completions`만 게이트웨이로 넘기는 경우에는 `30-ai-route.yaml`의 `AIGatewayRoute` 경로를 쓰면 됩니다.

cross-namespace 참조를 위해 `25-referencegrant-uplus-mready-model.yaml`의 `ReferenceGrant`도 함께 적용됩니다. 이 Grant는 `to.name` 없이 **`uplus-mready-model`의 모든 `Service`** 를 `envoy-ai-gateway-system`의 `HTTPRoute`가 참조할 수 있게 합니다(편의성↑, 노출 범위도 넓어지므로 운영 정책에 맞게 조정하세요).

이전에 `allow-psm-llm-gateway-to-fastapi-greet` 이름으로 적용해 둔 `ReferenceGrant`가 있다면, 중복이므로 `k delete referencegrant allow-psm-llm-gateway-to-fastapi-greet -n uplus-mready-model` 로 제거해도 됩니다.

### 3) 게이트웨이 접근 (내부망 기준)

Envoy Gateway가 만든 데이터플레인 Service를 확인합니다(환경에 따라 LoadBalancer/NodePort/ClusterIP가 다를 수 있습니다).

```bash
k get svc -n envoy-gateway-system \
  --selector=gateway.envoyproxy.io/owning-gateway-namespace=envoy-ai-gateway-system,gateway.envoyproxy.io/owning-gateway-name=psm-llm-gateway
```

### 4) 테스트 요청

아래는 `mlp-model-name: gpt-oss-120b` 헤더로 `uplus-mready-model/gpt-oss-120b:80`으로 라우팅되는 구성 기준입니다.

```bash
export GATEWAY_URL="http://<gateway-address-or-ip>"

curl -H "Content-Type: application/json" \
  -H "mlp-model-name: gpt-oss-120b" \
  -d '{
    "model": "gpt-oss-120b",
    "messages": [{"role":"system","content":"Hi."}]
  }' \
  "$GATEWAY_URL/v1/chat/completions"
```

### 운영 전 체크리스트(필수)

- `k8s/values.yaml`의 `controller.mcp.sessionEncryption.seed`를 **안전한 랜덤 값**으로 교체
- 사내 방화벽/프록시/이미지 레지스트리 정책에 맞게 imagePullSecrets, mirror 설정 반영
- 외부/사내망 노출 방식(LoadBalancer/Ingress/Gateway API) 확정 후 `Gateway` 인프라 설정 조정(플랫폼 표준 `EnvoyProxy`를 쓰는 경우 이 레포에서 별도 `EnvoyProxy`는 관리하지 않음)
- 클러스터에 `GatewayClass`가 이미 있으면 `k8s/gateway/00-gatewayclass.yaml` 적용을 생략하고, `10-gateway.yaml`의 `gatewayClassName`만 플랫폼 표준 이름으로 맞출 것
