# Traefik Gateway API PoC — Hướng dẫn cài đặt & cấu hình

Quy trình dưới đây đã được kiểm chứng trên **kind (Kubernetes v1.37.0)** với **Traefik v3.7.17**
(Helm chart `traefik/traefik` 42.x) và **Gateway API CRD v1.6.1 (experimental channel)**.
Mọi dịch vụ chạy nội bộ (ClusterIP) và chỉ truy cập qua `kubectl port-forward` — không expose ra ngoài.

> Nếu cluster của bạn **đã chạy Envoy Gateway**, đọc kỹ phần [Cùng tồn tại với Envoy](#cùng-tồn-tại-với-envoy-gateway)
> trước khi cài — không ghi đè CRD/GatewayClass của Envoy.

---

## 1. Yêu cầu công cụ

| Công cụ | Phiên bản đã dùng | Ghi chú |
|---|---|---|
| Docker | 29.8.2 | Runtime cho kind |
| kind | v0.33.0 | Cluster Kubernetes trong Docker |
| kubectl | v1.37.2 | |
| Helm | v3.22.1 | |
| curl | bất kỳ | Dùng cho smoke test |

Kiểm tra nhanh:

```bash
docker version --format '{{.Server.Version}}'
kind version && kubectl version --client && helm version --short
```

---

## 2. Tạo cluster (kind)

PoC cần map cổng 80/443 từ host vào node để Gateway nhận traffic. File cấu hình mẫu:

```yaml
# kind-traefik.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: traefik-poc
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - { containerPort: 30080, hostPort: 80,  protocol: TCP }
      - { containerPort: 30443, hostPort: 443, protocol: TCP }
  - role: worker
  - role: worker
```

```bash
kind create cluster --config kind-traefik.yaml
kubectl wait --for=condition=Ready nodes --all --timeout=90s
```

> Trên cluster thật (không phải kind) bỏ qua bước này và trỏ `kubectl` vào cluster có sẵn.

---

## 3. Cài Gateway API CRDs — BẮT BUỘC experimental channel

**Đây là điểm vướng quan trọng nhất.** Traefik v3.7 provider `kubernetesGateway` watch
`TLSRoute` và `BackendTLSPolicy` ở **API version `v1`**. Hai CRD này:

- **không** nằm trong *standard channel*;
- bản *experimental* cũ (v1.3.0) chỉ phục vụ chúng ở `v1alpha2`/`v1alpha3`.

→ Nếu cài thiếu/đúng version, GatewayClass kẹt ở `Accepted=Unknown` ("Waiting for controller")
và **không route nào được programmed**.

Cài **experimental channel v1.6.1** (phục vụ `v1`), dùng `--server-side` để tránh giới hạn
256KB của annotation `last-applied-configuration` trên CRD `httproutes`:

```bash
kubectl apply --server-side --force-conflicts \
  -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.6.1/experimental-install.yaml
```

Kiểm tra version đã phục vụ `v1`:

```bash
kubectl get crd backendtlspolicies.gateway.networking.k8s.io tlsroutes.gateway.networking.k8s.io \
  -o custom-columns='NAME:.metadata.name,VERSIONS:.spec.versions[*].name'
# Mong đợi: backendtlspolicies -> v1,v1alpha3 ; tlsroutes -> v1,v1alpha2,v1alpha3
```

---

## 4. Cài Traefik (Helm)

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
helm upgrade --install traefik-poc traefik/traefik \
  --namespace traefik-poc --create-namespace \
  -f helm/values.yaml
kubectl -n traefik-poc rollout status deployment/traefik-poc
```

### Giải thích `helm/values.yaml`

| Khoá | Ý nghĩa |
|---|---|
| `providers.kubernetesGateway.enabled: true` | Bật provider Gateway API (đọc GatewayClass/Gateway/HTTPRoute) |
| `providers.kubernetesCRD.enabled: true` | Bật CRD Traefik (Middleware...) — cần cho rate limit |
| `providers.kubernetesIngress.enabled: false` | Tắt Ingress cổ điển, chỉ dùng Gateway API |
| `service.spec.type: ClusterIP` | Không expose LoadBalancer; truy cập qua port-forward |
| `ports.web` / `ports.websecure` | Entrypoint 8000/8443, lộ ra 80/443 |
| `metrics.prometheus.*` | Bật metric Prometheus + `serviceMonitor` (label `release: traefik-monitoring`) |
| `accessLog.enabled/format: json` | Access log JSON |
| `api.dashboard: true`, `api.insecure: false` | Bật dashboard, không mở API insecure |
| `ingressRoute.dashboard` | Tạo IngressRoute cho dashboard trên entrypoint `web` |

> **Lưu ý schema chart 42.x** (khác bản cũ): dùng `service.spec.type` (không phải `service.type`)
> và `accessLog:` ở gốc (không phải `logs.access`). Sai schema sẽ bị Helm từ chối khi validate.

### Áp GatewayClass, Gateway, backend và lab đầu

```bash
kubectl apply -f base/backend.yaml      # echo-v1 / echo-v2
kubectl apply -f base/gateway.yaml      # GatewayClass + Gateway traefik-poc
kubectl apply -f labs/01-http-routing.yaml
```

Xác nhận đã sẵn sàng:

```bash
kubectl get gatewayclass traefik-poc \
  -o jsonpath='Accepted={.status.conditions[?(@.type=="Accepted")].status}{"\n"}'   # True
kubectl -n traefik-poc get gateway traefik-poc \
  -o jsonpath='Programmed={.status.conditions[?(@.type=="Programmed")].status}{"\n"}' # True
```

---

## 5. Lab & smoke test

```bash
kubectl apply -f labs/02-weighted.yaml -f labs/03-redirect.yaml \
              -f labs/04-headers.yaml -f labs/05-ratelimit.yaml

# port-forward entrypoint web rồi chạy smoke test
kubectl -n traefik-poc port-forward svc/traefik-poc 18080:80 &
BASE_URL=http://127.0.0.1:18080 bash tests/smoke.sh
```

Kiểm chứng mở rộng (bằng chứng ghi trong `docs/feature-matrix.csv`):

```bash
# Weighted split 90/10
for i in $(seq 1 200); do curl -s -H 'Host: weighted.traefik.test' \
  http://127.0.0.1:18080/ | grep -o 'echo-v[12]'; done | sort | uniq -c
# Rate limit (avg5/burst10) -> vài 200 rồi 429
for i in $(seq 1 40); do curl -s -o /dev/null -w '%{http_code}\n' \
  -H 'Host: ratelimit.traefik.test' http://127.0.0.1:18080/; done | sort | uniq -c
```

---

## 6. Dashboard + Prometheus + Grafana (tùy chọn)

Cài **sau** khi đã cài Gateway API CRDs (script đã lo thứ tự: cài kube-prometheus-stack trước
để có CRD `ServiceMonitor`, rồi mới upgrade Traefik):

```bash
bash scripts/install-monitoring.sh
bash scripts/monitoring-status.sh
```

### Truy cập (mỗi lệnh giữ chạy ở 1 terminal)

| Dịch vụ | port-forward | URL |
|---|---|---|
| Traefik Dashboard | `kubectl -n traefik-poc port-forward svc/traefik-poc 18081:80` | http://localhost:18081/dashboard/ (nhớ dấu `/` cuối) |
| Grafana | `kubectl -n traefik-monitoring port-forward svc/traefik-monitoring-grafana 3000:80` | http://localhost:3000 (user `admin`) |
| Prometheus | `kubectl -n traefik-monitoring port-forward svc/traefik-monitoring-kube-pr-prometheus 9090:9090` | http://localhost:9090 |

Mật khẩu Grafana:

```bash
kubectl -n traefik-monitoring get secret traefik-monitoring-grafana \
  -o jsonpath='{.data.admin-password}' | base64 -d; echo
```

> **Tên service Prometheus** thực tế là `traefik-monitoring-kube-pr-prometheus` (chart rút gọn
> `kube-prometheus` → `kube-pr`). Grafana dashboard: **Traefik Gateway PoC / Performance**.

### Dashboard matchRule

`helm/values.yaml` đặt `ingressRoute.dashboard.matchRule` chỉ theo path
(`PathPrefix(/dashboard) || PathPrefix(/api)`) để mở thẳng bằng `localhost` không cần sửa file hosts.
An toàn vì service là ClusterIP + chỉ qua port-forward nội bộ. Nếu muốn ràng buộc theo hostname,
đổi lại thành `Host(\`dashboard.traefik.test\`) && (PathPrefix(\`/dashboard\`) || PathPrefix(\`/api\`))`
và thêm `127.0.0.1 dashboard.traefik.test` vào file hosts của máy chạy trình duyệt.

---

## 7. Xử lý sự cố

| Triệu chứng | Nguyên nhân | Khắc phục |
|---|---|---|
| GatewayClass `Accepted=Unknown` ("Waiting for controller"); không route nào programmed | Thiếu CRD `TLSRoute`/`BackendTLSPolicy` ở version `v1` | Cài experimental channel v1.6.1 ([mục 3](#3-cài-gateway-api-crds--bắt-buộc-experimental-channel)) rồi `kubectl -n traefik-poc rollout restart deployment/traefik-poc` |
| Helm báo `additional properties 'type'/'logs' not allowed` | Dùng schema chart cũ | `service.spec.type`, `accessLog` ([mục 4](#giải-thích-helmvaluesyaml)) |
| `helm template` lỗi `You have to deploy monitoring.coreos.com/v1 first` | `serviceMonitor.enabled: true` nhưng chưa có CRD ServiceMonitor | Cài kube-prometheus-stack trước (`scripts/install-monitoring.sh` đã đúng thứ tự) |
| Dashboard trả `404` khi mở `localhost:18081/dashboard/` | matchRule yêu cầu Host header | Dùng matchRule theo path (mặc định hiện tại), hoặc gửi `-H 'Host: dashboard.traefik.test'` |
| `httproutes` CRD apply lỗi `metadata.annotations: Too long` | Giới hạn 256KB của client-side apply | Dùng `kubectl apply --server-side --force-conflicts` |
| Panel Grafana trống | Chưa có traffic hoặc tên metric/nhãn khác version | Sinh traffic; kiểm tra `up` và `/api/v1/label/__name__/values` trong Prometheus |

---

## Cùng tồn tại với Envoy Gateway

- Traefik dùng GatewayClass riêng `traefik-poc` (`controllerName: traefik.io/gateway-controller`);
  **không** sửa GatewayClass/Gateway/route của Envoy.
- Kiểm tra channel/version CRD Gateway API hiện có **trước** khi nâng — không hạ cấp CRD mà Envoy đang dùng:
  `kubectl get crd -l gateway.networking.k8s.io/bundle-version` và `kubectl get gatewayclasses`.
- Stack monitoring cài ở namespace riêng `traefik-monitoring`; nếu đã có Prometheus/Grafana dùng chung,
  bỏ qua stack này và chỉ đặt nhãn `ServiceMonitor` phù hợp trong `helm/values.yaml`.
- Chỉ `helm upgrade` release `traefik-poc` / `traefik-monitoring`; **không** `helm uninstall` release khác.

---

## Gỡ bỏ

```bash
helm uninstall traefik-poc -n traefik-poc
helm uninstall traefik-monitoring -n traefik-monitoring
kubectl delete ns traefik-poc traefik-monitoring
# Chỉ xoá CRD Gateway API nếu KHÔNG còn controller nào khác (vd Envoy) đang dùng.
kind delete cluster --name traefik-poc    # chỉ khi dùng kind
```
