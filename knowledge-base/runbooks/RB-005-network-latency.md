---
id: RB-005
title: Mạng tới một service bị chậm, mất gói hoặc bị cắt
services: [productcatalogservice, adservice, currencyservice, frontend, checkoutservice]
fault_type: network
severity: medium
chaos_experiment: NetworkChaos/delay, NetworkChaos/loss, NetworkChaos/partition
status: draft
sources: [S01, S02, S05, S06, S08, S12, S13]
---

# RB-005: Mạng tới một service bị chậm, mất gói hoặc bị cắt

## Tóm tắt
Kết nối mạng tới một service bị trễ (delay), mất gói (loss) hoặc bị cắt hẳn
(partition). Service vẫn chạy bình thường: **CPU, RAM, trạng thái pod đều
không đổi**. Không có Istio thì gần như không phát hiện được lỗi này, vì chỉ
còn độ trễ tổng của frontend.

Ví dụ dùng `productcatalogservice`.

## Cách tái hiện (chaos experiment)
[S08] Thêm 300ms trễ cho pod productcatalogservice:
```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: productcatalog-delay
spec:
  action: delay
  mode: all
  duration: '10m'
  selector:
    namespaces: [default]
    labelSelectors:
      app: productcatalogservice
  delay:
    latency: '300ms'
    jitter: '0ms'
    correlation: '100'
```
Biến thể: `action: loss` (mất gói), `action: partition` (cắt mạng).

## Triệu chứng: Metrics
| Tín hiệu | PromQL | Bình thường → khi lỗi |
|---|---|---|
| p99 độ trễ đo **phía gọi** [S01][S13] | `histogram_quantile(0.99, sum by (le, source_workload) (rate(istio_request_duration_milliseconds_bucket{reporter="source", destination_workload="productcatalogservice"}[1m])))` | Vài ms → tăng ít nhất khoảng bằng mức delay |
| p99 độ trễ đo **phía nhận** | Như trên, đổi `reporter="destination"` | **Dự kiến tăng ít hoặc không tăng** (xem giải thích dưới) |
| CPU, RAM của service [S06] | `rate(container_cpu_usage_seconds_total{container="server", pod=~"productcatalogservice.*"}[1m])` | **Không đổi** |
| Pod Ready [S05] | `kube_pod_container_status_ready{pod=~"productcatalogservice.*"}` | **Vẫn = 1** |
| Mã lỗi (chỉ khi `partition`, hoặc `loss` nặng) [S02] | `sum(rate(istio_requests_total{reporter="source", destination_workload="productcatalogservice", grpc_response_status=~"14\|4"}[1m]))` | ≈ 0 → tăng |

**Dấu hiệu quan trọng nhất: độ trễ phía gọi lệch xa độ trễ phía nhận.**
Sidecar phía nhận nằm *bên trong* pod bị tiêm lỗi nên chỉ đo thời gian service
xử lý. Sidecar phía gọi đo cả thời gian đi qua mạng. Khoảng chênh lệch giữa hai
con số chính là phần trễ do mạng. **Điều này là suy luận, cần xác nhận khi chạy
thí nghiệm thật** rồi cập nhật lại runbook.

## Triệu chứng: Logs
- `delay`: thường **không có log lỗi**. frontend `request complete` có `http.resp.took_ms` tăng (`src/frontend/middleware.go:75`) [S12].
- `partition` / `loss` nặng: bên gọi log `code = Unavailable` (giống RB-002).

### Trường hợp đặc biệt: adservice
frontend chỉ chờ adservice **100ms** (`src/frontend/rpc.go:120`) [S12]. Delay
tới adservice > 100ms → mọi lời gọi quá hạn (`DEADLINE_EXCEEDED`, mã 4 [S02]),
frontend log `warn`: `failed to retrieve ads` (`src/frontend/handlers.go:530`).
Trang vẫn trả 200, chỉ mất quảng cáo. Đây là lỗi "âm thầm".
```logql
sum(count_over_time({app="frontend"} |= "failed to retrieve ads" [1m]))
```

## Lan truyền lỗi
```
productcatalogservice (mạng chậm, bản thân vẫn khỏe)
   ├─► frontend: trang chủ, trang sản phẩm chậm
   ├─► recommendationservice: chậm theo → trang sản phẩm chậm thêm
   └─► checkoutservice: đặt hàng chậm
```

## Phân biệt với lỗi khác
| Dễ nhầm với | Dấu hiệu để tách |
|---|---|
| RB-004: CPU quá tải | CPU quá tải → CPU chạm giới hạn, throttle cao, độ trễ **phía nhận cũng tăng**. Mạng chậm → CPU bình thường, chủ yếu độ trễ **phía gọi** tăng. |
| RB-002: service chết | Cả hai đều có thể ra mã 14 (khi partition). Service chết → pod Ready = 0. Partition → pod vẫn Ready = 1. |
| Service phía sau chậm | Nếu productcatalogservice chậm vì chính nó gọi service khác chậm thì độ trễ phía nhận cũng tăng. productcatalogservice không gọi service nào khác ([topology.md](../topology.md)), nên với ví dụ này có thể loại trường hợp đó. |

## Các bước xác minh
```bash
kubectl get networkchaos -A
kubectl get pods -l app=productcatalogservice      # vẫn Running, Ready
kubectl top pod -l app=productcatalogservice       # CPU, RAM bình thường
# So sánh độ trễ reporter="source" và reporter="destination" trên Grafana
```

## Khắc phục
- **Ngay:** dừng thí nghiệm chaos; kiểm tra network policy, node, CNI.
- **Lâu dài:** đặt timeout và retry hợp lý cho gRPC client (Istio `VirtualService` hỗ trợ timeout/retries); alert theo độ trễ p99 phía `reporter="source"`.

## Nguồn
- [S01] Istio Standard Metrics: `istio_request_duration_milliseconds`, label `reporter`
- [S02] gRPC Status Codes: mã 4 `DEADLINE_EXCEEDED`, mã 14 `UNAVAILABLE`
- [S05] kube-state-metrics: `kube_pod_container_status_ready`
- [S06] cAdvisor: metric CPU
- [S08] Chaos Mesh NetworkChaos: `delay`, `loss`, `partition`; YAML mẫu
- [S12] Mã nguồn Online Boutique: timeout 100ms, log `took_ms`, topology
- [S13] Prometheus `histogram_quantile`
