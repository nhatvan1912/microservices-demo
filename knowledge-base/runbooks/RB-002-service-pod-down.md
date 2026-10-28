---
id: RB-002
title: Một service bị chết hoặc không có pod nào sẵn sàng
services: [productcatalogservice, currencyservice, cartservice, checkoutservice, paymentservice, shippingservice, emailservice, recommendationservice, adservice]
fault_type: pod_down
severity: high
chaos_experiment: PodChaos/pod-kill, PodChaos/pod-failure, PodChaos/container-kill
status: draft
sources: [S01, S02, S05, S07, S12]
---

# RB-002: Một service bị chết (không có pod sẵn sàng)

## Tóm tắt
Pod của một service bị xóa, bị treo hoặc container bị kill. Trong khoảng thời
gian chưa có pod mới sẵn sàng, mọi lời gọi tới service đó thất bại với mã gRPC
**14 `UNAVAILABLE`**. Mức ảnh hưởng tới người dùng phụ thuộc vào service nào
chết, xem bảng trong [topology.md](../topology.md).

Ví dụ trong runbook dùng `productcatalogservice`. Với service khác chỉ cần đổi tên.

## Cách tái hiện (chaos experiment)
[S07] có ba kiểu:
- `pod-kill`: xóa pod một lần. Deployment tạo pod mới, lỗi kéo dài vài giây tới vài chục giây.
- `pod-failure`: pod không hoạt động suốt `duration`. Dùng để có lỗi kéo dài, dễ quan sát.
- `container-kill`: kill một container trong pod. Pod giữ nguyên tên, số lần restart tăng.

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: productcatalog-failure
spec:
  action: pod-failure
  mode: all
  duration: '3m'
  selector:
    namespaces: [default]
    labelSelectors:
      app: productcatalogservice
```

## Triệu chứng: Metrics
| Tín hiệu | PromQL | Bình thường → khi lỗi |
|---|---|---|
| Bên gọi nhận mã 14 [S01][S02] | `sum by (source_workload) (rate(istio_requests_total{reporter="source", destination_workload="productcatalogservice", grpc_response_status="14"}[1m]))` | ≈ 0 → tăng |
| Lưu lượng **phía nhận** về 0 | `sum(rate(istio_requests_total{reporter="destination", destination_workload="productcatalogservice"}[1m]))` | Ổn định → ≈ 0 |
| Container không sẵn sàng [S05] | `sum(kube_pod_container_status_ready{namespace="default", pod=~"productcatalogservice.*", container="server"})` | ≥ 1 → 0 |
| Số lần restart (khi `container-kill` hoặc crash) [S05] | `increase(kube_pod_container_status_restarts_total{namespace="default", pod=~"productcatalogservice.*"}[5m])` | 0 → > 0 |
| frontend lỗi 5xx | `sum(rate(istio_requests_total{reporter="destination", destination_workload="frontend", response_code=~"5.."}[1m]))` | ≈ 0 → tăng (nếu service chết là service "nặng") |

**Chú ý `reporter`:** khi service đích chết, sidecar của nó cũng chết theo, nên
lỗi **chỉ được ghi ở phía gọi** (`reporter="source"`). Truy vấn
`reporter="destination"` sẽ thấy lưu lượng về 0 chứ không thấy lỗi.

**Chú ý `pod-kill`:** pod bị xóa và thay bằng pod mới tên khác, nên
`kube_pod_container_status_restarts_total` của pod mới bắt đầu từ 0, **không**
tăng. Dấu hiệu là tên pod thay đổi và tuổi pod (AGE) rất nhỏ.

## Triệu chứng: Logs
Go client ghi lỗi gRPC theo dạng `rpc error: code = Unavailable desc = ...`.

- **frontend**: `message="request error"`, trường `error` có
  `could not retrieve products: rpc error: code = Unavailable ...`
  (`src/frontend/handlers.go:69`) [S12]
- **checkoutservice**: `failed to prepare order` / `failed to get product #...`
  (`src/checkoutservice/main.go:296, 346`) [S12]
- **Service bị chết:** log bị ngắt. Khi pod mới lên sẽ có log khởi động lại.

```logql
sum by (app) (count_over_time({namespace="default"} |= "code = Unavailable" [1m]))
```

## Lan truyền lỗi
Xem [topology.md](../topology.md). Tóm tắt:
- **Hỏng nặng (trang chủ 500):** productcatalogservice, currencyservice, cartservice
- **Hỏng vừa (chỉ đặt hàng):** checkoutservice, paymentservice, shippingservice
- **Gần như không thấy:** adservice, recommendationservice, emailservice. frontend chỉ log mức `warn`, trang vẫn trả 200. **Không có lỗi 5xx**, nên alert theo tỉ lệ lỗi frontend sẽ không bắt được. Phải theo dõi mã 14 phía `reporter="source"`.

## Phân biệt với lỗi khác
| Dễ nhầm với | Dấu hiệu để tách |
|---|---|
| RB-003: OOMKilled | OOM → `kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}` = 1, RAM tăng dần trước khi chết, restart tăng nhưng tên pod giữ nguyên. |
| RB-005: mạng bị cắt (partition) | Partition → bên gọi cũng nhận mã 14, **nhưng pod đích vẫn Ready = 1** và vẫn sống. |
| RB-001: Redis lỗi | Redis lỗi → mã **9**, không phải 14. |
| Nhiều service lỗi cùng lúc | Tìm service ở **cuối chuỗi gọi** (theo topology) có Ready = 0. Các service phía trên chỉ là nạn nhân. |

## Các bước xác minh
```bash
kubectl get pods -l app=productcatalogservice -o wide   # READY, RESTARTS, AGE
kubectl describe pod -l app=productcatalogservice       # Events, Last State
kubectl get events --sort-by=.lastTimestamp | tail -20
kubectl get podchaos -A
```

## Khắc phục
- **Ngay:** dừng thí nghiệm chaos; kiểm tra Deployment đã tạo lại pod chưa.
- **Lâu dài:** chạy ít nhất 2 replica cho các service "nặng"; thêm `PodDisruptionBudget`; frontend nên có cơ chế xuống cấp mềm (vẫn hiện trang khi currency lỗi).

## Nguồn
- [S01] Istio Standard Metrics: `istio_requests_total`, label `reporter`, `grpc_response_status`
- [S02] gRPC Status Codes: mã 14 `UNAVAILABLE`
- [S05] kube-state-metrics: ready, restarts
- [S07] Chaos Mesh PodChaos: `pod-kill`, `pod-failure`, `container-kill`
- [S12] Mã nguồn Online Boutique: các dòng đã ghi ở trên
