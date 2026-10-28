---
id: RB-001
title: Redis (redis-cart) không truy cập được, giỏ hàng hỏng
services: [redis-cart, cartservice, frontend, checkoutservice]
fault_type: dependency_down
severity: high
chaos_experiment: PodChaos/pod-failure (redis-cart) hoặc NetworkChaos/partition (cartservice ↔ redis-cart)
status: draft
sources: [S01, S02, S05, S07, S08, S12]
---

# RB-001: Redis (redis-cart) không truy cập được

## Tóm tắt
cartservice lưu giỏ hàng trong Redis (`redis-cart:6379`). Khi Redis chết hoặc
bị cắt mạng, mọi thao tác giỏ hàng lỗi. Vì **trang chủ cũng đọc giỏ hàng**, người
dùng thấy lỗi 500 ngay khi vào trang chủ, không chỉ khi thêm hàng.

Điểm khó của lỗi này: **pod cartservice vẫn báo khỏe (Ready)**, nên nhìn trạng
thái pod sẽ không thấy cartservice có vấn đề. Phải nhìn mã lỗi gRPC và log.

## Cách tái hiện (chaos experiment)
Làm Redis ngừng hoạt động 5 phút [S07]:
```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: redis-cart-failure
spec:
  action: pod-failure
  mode: one
  duration: '5m'
  selector:
    namespaces: [default]
    labelSelectors:
      app: redis-cart
```
Biến thể: cắt mạng giữa cartservice và redis-cart bằng `NetworkChaos` với
`action: partition` [S08]. Khi đó pod Redis vẫn khỏe, chỉ là không ai tới được.

## Triệu chứng: Metrics
| Tín hiệu | PromQL | Bình thường → khi lỗi |
|---|---|---|
| cartservice trả mã lỗi gRPC 9 (`FAILED_PRECONDITION`) [S01][S02] | `sum(rate(istio_requests_total{reporter="destination", destination_workload="cartservice", grpc_response_status="9"}[1m]))` | ≈ 0 → tăng mạnh |
| Tỉ lệ lỗi 5xx của frontend | `sum(rate(istio_requests_total{reporter="destination", destination_workload="frontend", response_code=~"5.."}[1m])) / sum(rate(istio_requests_total{reporter="destination", destination_workload="frontend"}[1m]))` | ≈ 0 → gần như mọi trang đều lỗi |
| Kết nối TCP cartservice → Redis [S01] | `sum(rate(istio_tcp_connections_opened_total{reporter="source", source_workload="cartservice", destination_workload="redis-cart"}[1m]))` | Ổn định → thay đổi bất thường (client Redis kết nối lại liên tục hoặc không kết nối được) |
| Container Redis không sẵn sàng [S05] (chỉ khi dùng `pod-failure`) | `kube_pod_container_status_ready{namespace="default", container="redis"}` | 1 → 0 |
| cartservice vẫn Ready | `kube_pod_container_status_ready{namespace="default", container="server", pod=~"cartservice.*"}` | Vẫn = 1 (xem giải thích bên dưới) |

Độ trễ của cartservice **có thể** tăng trước khi trả lỗi, vì client Redis chờ
hết thời gian kết nối. Cần đo khi chạy thật để ghi số cụ thể.

## Triệu chứng: Logs
Câu lỗi gốc do cartservice tạo ra
(`src/cartservice/src/cartstore/RedisCartStore.cs:64, 79, 102`) [S12]:
```
Can't access cart storage. <chi tiết exception của thư viện Redis>
```
cartservice trả câu này về cho bên gọi qua mã gRPC `FailedPrecondition`.
Bên gọi ghi lại như sau:

- **frontend** (`src/frontend/handlers.go:74`, hàm `renderHTTPError` ở `handlers.go:537`):
  log JSON mức `error`, `message="request error"`, trường `error` chứa
  `could not retrieve cart: rpc error: code = FailedPrecondition desc = Can't access cart storage...`
  Các câu tương tự: `failed to add to cart` (`handlers.go:232`), `failed to empty cart` (`handlers.go:244`).
- **checkoutservice** (`src/checkoutservice/main.go:327`): `failed to get user cart during checkout`.

LogQL [S14] (tên label `app` tùy cấu hình Promtail):
```logql
sum(count_over_time({app="frontend"} |= "Can't access cart storage" [1m]))
```

## Lan truyền lỗi
```
redis-cart  ✖
   └─► cartservice (trả gRPC 9, pod vẫn Ready)
          ├─► frontend: trang chủ, trang sản phẩm, giỏ hàng → HTTP 500
          └─► checkoutservice: đặt hàng lỗi → frontend 500
```

## Phân biệt với lỗi khác
| Dễ nhầm với | Dấu hiệu để tách |
|---|---|
| RB-002: chính cartservice chết | cartservice chết → bên gọi nhận mã **14 `UNAVAILABLE`**, pod cartservice Ready = 0. Redis lỗi → mã **9 `FAILED_PRECONDITION`**, log có `Can't access cart storage`, pod cartservice vẫn Ready. |
| RB-005: mạng chậm tới cartservice | Mạng chậm → độ trễ tăng, thường không có mã 9, không có câu `Can't access cart storage`. |
| RB-003: Redis bị OOMKilled | Cùng triệu chứng phía cartservice. Kiểm tra `kube_pod_container_status_last_terminated_reason{container="redis", reason="OOMKilled"}`. Nếu có thì nguyên nhân gốc là hết RAM (Redis giới hạn 256Mi, `kubernetes-manifests/cartservice.yaml:133-134`). |

**Vì sao cartservice vẫn Ready:** health check của cartservice gọi `Ping()`,
nhưng hàm này luôn trả `true`, không thật sự kiểm tra Redis
(`src/cartservice/src/cartstore/RedisCartStore.cs:105-115`,
`src/cartservice/src/services/HealthCheckService.cs`) [S12]. Vì vậy Kubernetes
không restart cartservice, và nhìn trạng thái pod sẽ không phát hiện ra lỗi.

## Các bước xác minh
```bash
kubectl get pods -l app=redis-cart              # Redis còn chạy không?
kubectl describe pod -l app=redis-cart          # Lý do bị kill (OOMKilled?)
kubectl logs deploy/frontend | grep "Can't access cart storage" | tail
kubectl get networkchaos,podchaos -A            # Có thí nghiệm chaos đang chạy không?
```

## Khắc phục
- **Ngay:** khôi phục pod Redis (xóa thí nghiệm chaos, hoặc `kubectl rollout restart deploy/redis-cart`).
- **Lâu dài:** cho health check của cartservice thật sự ping Redis; dùng Redis có bản sao (Memorystore, Redis Sentinel); thêm alert riêng cho mã gRPC 9 của cartservice.

## Nguồn
- [S01] Istio Standard Metrics: tên metric và label
- [S02] gRPC Status Codes: ý nghĩa mã 9 và 14
- [S05] kube-state-metrics: `kube_pod_container_status_ready`
- [S07] Chaos Mesh PodChaos, [S08] Chaos Mesh NetworkChaos
- [S12] Mã nguồn Online Boutique: các file và dòng đã ghi ở trên
- [S14] Grafana Loki: cú pháp LogQL
