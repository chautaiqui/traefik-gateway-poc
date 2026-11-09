# Traefik Gateway API PoC — 80-feature comparison

A non-invasive Traefik OSS PoC for a Kubernetes cluster already running Envoy Gateway.

## Scope and evidence
- `docs/feature-matrix.csv`: 80 Envoy v1.9 feature names, P0–P3 priority, preliminary implementation *candidates*. **No feature is marked passed.**
- `tests/smoke.sh`: executable HTTP route, weighted split, redirect, header mutation, rate limit smoke checks. Tests are not equivalent to complete feature conformance.
- No modification to existing Envoy GatewayClass, Gateway or routes.
- Traefik-specific CRDs and external dependencies are explicitly separated from native Gateway API.

## Prerequisites
Kubernetes, kubectl, Helm 3, curl; network access to Helm chart repository. Gateway API CRDs must already be installed and compatible with Traefik version. Verify CRD versions before upgrading them; do not blindly overwrite Envoy's CRDs.

## Install
```bash
kubectl get gatewayclasses
kubectl get crd gateways.gateway.networking.k8s.io -o yaml | head
helm repo add traefik https://traefik.github.io/charts
helm repo update
helm upgrade --install traefik-poc traefik/traefik --namespace traefik-poc --create-namespace -f helm/values.yaml
kubectl apply -f base/backend.yaml
kubectl apply -f base/gateway.yaml
kubectl apply -f labs/01-http-routing.yaml
kubectl -n traefik-poc rollout status deployment/traefik-poc
kubectl -n traefik-poc port-forward svc/traefik-poc 18080:80
# in another shell
curl -i -H 'Host: echo.traefik.test' http://127.0.0.1:18080/
```

## Labs
```bash
kubectl apply -f labs/02-weighted.yaml
kubectl apply -f labs/03-redirect.yaml
kubectl apply -f labs/04-headers.yaml
kubectl apply -f labs/05-ratelimit.yaml
BASE_URL=http://127.0.0.1:18080 bash tests/smoke.sh
```

Run tests only after their corresponding manifests are applied. For production-like benchmarking, use a dedicated load generator, a consistent network path, separate exposed ports and identical backend resource limits; avoid `kubectl port-forward` for throughput benchmarking.

## Scoring
Each feature starts NOT_VERIFIED. Only change `result` to PASS after storing the manifest, command, actual response, controller/version, and logs. Use PARTIAL, FAIL, NOT_SUPPORTED, or BLOCKED for other outcomes. Do not count candidates as coverage.

## Cautions
- The Helm chart and Traefik version are not pinned here: pin chart digest/version and image before formal comparison.
- Traefik CRD middleware integration requires the Kubernetes CRD provider.
- Some advanced functions may require Traefik Hub or a separate service; OSS-only support must be verified per feature.
- Port-forward is only for functional smoke tests.
- Keep existing Envoy resources intact.

Official docs: https://doc.traefik.io/traefik/reference/routing-configuration/kubernetes/gateway-api/
