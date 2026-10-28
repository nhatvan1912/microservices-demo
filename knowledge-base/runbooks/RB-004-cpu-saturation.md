---
id: RB-004
title: Service quá tải CPU, bị throttle, phản hồi chậm
services: [currencyservice, productcatalogservice, checkoutservice, frontend]
fault_type: cpu
severity: medium
chaos_experiment: StressChaos/cpu
status: draft
sources: [S01, S04, S06, S09, S10, S12, S13, S14]
---

# RB-004: Service quá tải CPU (CPU throttling)

## Tóm tắt
Khi container dùng CPU chạm giới hạn (`limits.cpu`), kernel **bóp CPU
(throttle)** chứ không kill container [S04]. Kết quả là service **chậm đi
nhưng thường không báo lỗi**. Đây là lỗi thuộc nhóm "saturation" trong bốn tín
hiệu vàng của Google SRE [S10].

Ví dụ dùng `currencyservice` (giới hạn CPU 200m,
`kubernetes-manifests/currencyservice.yaml:66`) [S12]. Service này bị gọi
**nhiều lần trên mỗi lần tải trang chủ** (đổi tiền cho từng sản phẩm,
`src/frontend/handlers.go:86`), nên khi nó chậm, trang chủ chậm rất rõ.

## Cách tái hiện (chaos experiment)
[S09] Tổng tải CPU = `workers` × `load`%.
```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: currency-cpu
spec:
  mode: one
  duration: '10m'
  selector:
    namespaces: [default]
    labelSelectors:
      app: currencyservice
  stressors:
    cpu:
      workers: 2
      load: 100
```
Tiến trình gây tải chạy cùng container nên dùng chung giới hạn 200m với service.
Cần xác nhận khi chạy thử.

## Triệu chứng: Metrics
| Tín hiệu | PromQL | Bình thường → khi lỗi |
|---|---|---|
| CPU đang dùng (đơn vị: core) [S06] | `sum(rate(container_cpu_usage_seconds_total{namespace="default", container="server", pod=~"currencyservice.*"}[1m]))` | Thấp → ≈ 0.2 (chạm giới hạn 200m) |
| Tỉ lệ bị throttle [S06] | `sum(rate(container_cpu_cfs_throttled_periods_total{namespace="default", container="server", pod=~"currencyservice.*"}[1m])) / sum(rate(container_cpu_cfs_periods_total{namespace="default", container="server", pod=~"currencyservice.*"}[1m]))` | ≈ 0 → cao (ví dụ > 0.5) |
| p99 độ trễ của currencyservice [S01][S13] | `histogram_quantile(0.99, sum by (le) (rate(istio_request_duration_milliseconds_bucket{reporter="destination", destination_workload="currencyservice"}[1m])))` | Vài ms → tăng nhiều lần |
| p99 độ trễ của frontend | Như trên, đổi `destination_workload="frontend"` | Tăng theo |
| Tỉ lệ lỗi | `istio_requests_total{grpc_response_status!="0"}` | **Thường không đổi** |

`container="server"` để loại container `istio-proxy` (sidecar) ra khỏi phép tính.

## Triệu chứng: Logs
- **Không có log lỗi** đặc trưng. Đây là lý do lỗi CPU khó nhận ra nếu chỉ đọc log.
- frontend ghi thời gian xử lý mỗi request: `message="request complete"`, trường
  `http.resp.took_ms` (`src/frontend/middleware.go:75-77`, mức log debug đang
  bật sẵn ở `src/frontend/main.go:94`) [S12]. Có thể dùng làm số liệu độ trễ dự
  phòng từ Loki [S14]:
```logql
quantile_over_time(0.99,
  {app="frontend"} |= "request complete" | json | unwrap http_resp_took_ms [1m])
```

## Lan truyền lỗi
```
currencyservice (CPU chạm giới hạn → chậm)
   ├─► frontend: trang chủ, trang sản phẩm, giỏ hàng chậm
   └─► checkoutservice: đặt hàng chậm
```
Nếu service bị quá tải có bên gọi đặt timeout ngắn, chậm sẽ biến thành lỗi.
Ví dụ frontend chỉ chờ adservice **100ms** (`src/frontend/rpc.go:120`) [S12]:
CPU adservice cao → quảng cáo biến mất, log `warn` `failed to retrieve ads`.

## Phân biệt với lỗi khác
| Dễ nhầm với | Dấu hiệu để tách |
|---|---|
| RB-005: mạng chậm | Mạng chậm → **CPU và throttle bình thường**, nhưng độ trễ tăng. CPU quá tải → CPU chạm giới hạn và tỉ lệ throttle cao **trong chính service chậm**. |
| Lưu lượng tăng đột biến (traffic spike) | Traffic spike → `rate(istio_requests_total)` tăng mạnh **trước** khi CPU tăng. CPU stress → lưu lượng không đổi mà CPU vẫn tăng. |
| RB-003: hết RAM | Hết RAM → container bị kill, restart tăng. CPU quá tải → không restart. |
| Service phía sau chậm | Nếu service A chậm vì gọi B chậm thì CPU của A bình thường. Tìm service **chậm mà CPU cao** ở cuối chuỗi gọi. |

## Các bước xác minh
```bash
kubectl top pod -l app=currencyservice --containers
kubectl get stresschaos -A
kubectl exec deploy/currencyservice -c server -- top -b -n1   # tiến trình lạ ngốn CPU? (nếu image có lệnh top)
```

## Khắc phục
- **Ngay:** dừng thí nghiệm chaos; tạm tăng `limits.cpu` hoặc thêm replica.
- **Lâu dài:** bật HorizontalPodAutoscaler theo CPU; alert theo tỉ lệ throttle chứ không chỉ theo % CPU.

## Nguồn
- [S01] Istio Standard Metrics: `istio_request_duration_milliseconds`
- [S04] Kubernetes – Resource Management: CPU limit được thực thi bằng throttling
- [S06] cAdvisor: `container_cpu_usage_seconds_total`, `container_cpu_cfs_*`
- [S09] Chaos Mesh StressChaos: `workers`, `load`
- [S10] Google SRE Book: tín hiệu "saturation"
- [S12] Mã nguồn Online Boutique: giới hạn CPU, log `took_ms`, timeout 100ms
- [S13] Prometheus `histogram_quantile`
- [S14] Grafana Loki: `json`, `unwrap`, `quantile_over_time`
