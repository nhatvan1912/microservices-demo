---
id: RB-003
title: Container hết bộ nhớ và bị OOMKilled
services: [currencyservice, cartservice, checkoutservice, redis-cart]
fault_type: memory
severity: high
chaos_experiment: StressChaos/memory
status: draft
sources: [S03, S04, S05, S06, S09, S12]
---

# RB-003: Container hết bộ nhớ (OOMKilled)

## Tóm tắt
Container dùng RAM vượt giới hạn (`limits.memory`) thì bị kernel kill với lý do
**`OOMKilled`**, exit code **137**, sau đó kubelet khởi động lại [S03]. Nếu RAM
tiếp tục tăng, quá trình kill → khởi động lại lặp đi lặp lại, có thể dẫn tới
`CrashLoopBackOff`.

Các service của Boutique có giới hạn RAM rất nhỏ, ví dụ currencyservice,
cartservice và checkoutservice chỉ có **128Mi**
(`kubernetes-manifests/currencyservice.yaml:65-67`, `cartservice.yaml:56-58`,
`checkoutservice.yaml:73-75`) [S12], nên rất dễ bị OOM.

Ví dụ trong runbook dùng `currencyservice`.

## Cách tái hiện (chaos experiment)
Dùng StressChaos chiếm thêm RAM trong container [S09]. `time` làm RAM tăng
dần (giống rò rỉ bộ nhớ), bỏ `time` thì tăng ngay lập tức.
```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: currency-memory
spec:
  mode: one
  duration: '10m'
  selector:
    namespaces: [default]
    labelSelectors:
      app: currencyservice
  stressors:
    memory:
      workers: 1
      size: '200MB'
      time: '5min'
```

## Triệu chứng: Metrics
| Tín hiệu | PromQL | Bình thường → khi lỗi |
|---|---|---|
| RAM so với giới hạn [S06][S05] | `max(container_memory_working_set_bytes{namespace="default", container="server", pod=~"currencyservice.*"}) / max(kube_pod_container_resource_limits{namespace="default", container="server", pod=~"currencyservice.*", resource="memory"})` | Thấp, ổn định → tăng dần tới ≈ 1 rồi rơi về thấp (hình răng cưa) |
| Lý do kill gần nhất [S05] | `kube_pod_container_status_last_terminated_reason{namespace="default", reason="OOMKilled"}` | không có → = 1 |
| Sự kiện OOM [S06] | `increase(container_oom_events_total{namespace="default", container="server"}[5m])` | 0 → > 0 |
| Số lần restart [S05] | `increase(kube_pod_container_status_restarts_total{namespace="default", pod=~"currencyservice.*"}[10m])` | 0 → > 0, tăng dần |
| Bên gọi nhận mã 14 trong lúc container khởi động lại | Như RB-002 | Lỗi theo từng đợt ngắn |

**Gợi ý cho phần phát hiện bất thường:**
- RAM tăng **từ từ** (rò rỉ bộ nhớ) → **EWMA** phát hiện tốt hơn Modified Z-Score.
- RAM tăng **đột ngột** → Modified Z-Score phát hiện được.
- RAM cao kèm restart và lỗi 14 cùng lúc → Isolation Forest (nhiều chiều) có lợi thế.

## Triệu chứng: Logs
- **Service bị OOM thường không ghi log lỗi nào**, vì bị kernel kill ngay bằng
  SIGKILL. Log chỉ bị ngắt, rồi xuất hiện lại log khởi động. Với currencyservice:
  `Starting gRPC server on port ...` và `CurrencyService gRPC server started on port ...`
  (`src/currencyservice/server.js:183, 192`) [S12].
- **Bên gọi** ghi lỗi `code = Unavailable` trong lúc container khởi động lại
  (frontend: `could not retrieve currencies`, `src/frontend/handlers.go:64`).

```logql
sum(count_over_time({app="currencyservice"} |= "gRPC server started" [5m]))
```
Nếu số lần khởi động > 0 mà không có thao tác deploy nào, đó là dấu hiệu bị restart.

## Lan truyền lỗi
Giống RB-002 nhưng **theo từng đợt**: mỗi lần container bị kill có vài giây lỗi,
sau đó hồi phục, rồi lặp lại khi RAM lại đầy.

## Phân biệt với lỗi khác
| Dễ nhầm với | Dấu hiệu để tách |
|---|---|
| RB-002: pod bị kill | pod-kill → không có RAM tăng trước đó, tên pod đổi. OOM → RAM chạm giới hạn ngay trước khi chết, `reason="OOMKilled"`, exit code 137, tên pod giữ nguyên. |
| RB-004: CPU quá tải | CPU quá tải → service chậm nhưng **không bị kill**, không restart [S04]. |
| Node hết RAM (bị evict) | Pod bị evict có lý do `Evicted` (xem `kubectl describe pod`), không phải `OOMKilled`. |

## Các bước xác minh
```bash
kubectl get pods -l app=currencyservice                # RESTARTS tăng?
kubectl describe pod -l app=currencyservice | grep -A5 "Last State"
#   Mong đợi: Reason: OOMKilled, Exit Code: 137
kubectl top pod -l app=currencyservice                 # RAM hiện tại
kubectl get stresschaos -A
```

## Khắc phục
- **Ngay:** dừng thí nghiệm chaos; tạm tăng `limits.memory`.
- **Lâu dài:** tìm chỗ rò rỉ bộ nhớ (profiling); đặt `limits` dựa trên số liệu sử dụng thực tế; alert khi RAM > 90% giới hạn để báo trước khi bị kill.

## Nguồn
- [S03] Kubernetes – Assign Memory Resources: OOMKilled, exit code 137, tự restart
- [S04] Kubernetes – Resource Management: CPU bị throttle chứ không bị kill
- [S05] kube-state-metrics: `last_terminated_reason`, `restarts_total`
- [S06] cAdvisor: `container_memory_working_set_bytes`, `container_oom_events_total`
- [S09] Chaos Mesh StressChaos: tham số `size`, `time`
- [S12] Mã nguồn Online Boutique: giới hạn RAM, log khởi động
