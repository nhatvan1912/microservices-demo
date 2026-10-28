# Nguồn tham khảo

Mọi link dưới đây đã được mở và kiểm tra nội dung ngày **2026-10-28**.
Runbook trích dẫn bằng mã `[Sxx]`.

| Mã | Nguồn | Dùng để xác minh điều gì |
|---|---|---|
| S01 | Istio – Standard Metrics<br>https://istio.io/latest/docs/reference/config/metrics/ | Tên metric `istio_requests_total`, `istio_request_duration_milliseconds`, `istio_tcp_connections_opened_total`, `istio_tcp_connections_closed_total`; label `response_code`, `grpc_response_status`, `source_workload`, `destination_workload` |
| S02 | gRPC – Status Codes<br>https://grpc.io/docs/guides/status-codes/ | Ý nghĩa mã `DEADLINE_EXCEEDED (4)`, `FAILED_PRECONDITION (9)`, `UNAVAILABLE (14)` |
| S03 | Kubernetes – Assign Memory Resources to Containers and Pods<br>https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/ | Container vượt giới hạn RAM bị kill với lý do `OOMKilled`, exit code 137, kubelet tự khởi động lại |
| S04 | Kubernetes – Resource Management for Pods and Containers<br>https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/ | Vượt giới hạn CPU thì bị **throttle** (bóp CPU), không bị kill |
| S05 | kube-state-metrics – Pod Metrics<br>https://github.com/kubernetes/kube-state-metrics/blob/main/docs/metrics/workload/pod-metrics.md | `kube_pod_container_status_restarts_total`, `kube_pod_container_status_last_terminated_reason`, `kube_pod_container_status_ready` |
| S06 | cAdvisor – Prometheus metrics<br>https://github.com/google/cadvisor/blob/master/docs/storage/prometheus.md | `container_cpu_usage_seconds_total`, `container_cpu_cfs_periods_total`, `container_cpu_cfs_throttled_periods_total`, `container_memory_working_set_bytes`, `container_oom_events_total` |
| S07 | Chaos Mesh – Simulate Pod Faults<br>https://chaos-mesh.org/docs/simulate-pod-chaos-on-kubernetes/ | Các kiểu lỗi `pod-failure`, `pod-kill`, `container-kill` |
| S08 | Chaos Mesh – Simulate Network Faults<br>https://chaos-mesh.org/docs/simulate-network-chaos-on-kubernetes/ | Các kiểu lỗi mạng `delay`, `loss`, `partition`, ...; YAML mẫu |
| S09 | Chaos Mesh – Simulate Stress Scenarios<br>https://chaos-mesh.org/docs/simulate-heavy-stress-on-kubernetes/ | Tiêm tải CPU (`workers`, `load`) và RAM (`size`, `time`) |
| S10 | Google SRE Book – Monitoring Distributed Systems<br>https://sre.google/sre-book/monitoring-distributed-systems/ | Bốn tín hiệu vàng: latency, traffic, errors, saturation |
| S11 | Awesome Prometheus Alerts<br>https://samber.github.io/awesome-prometheus-alerts/rules | Bộ rule cảnh báo mẫu cho Kubernetes (46 rule) và Istio (11 rule) để tham khảo ngưỡng |
| S12 | Mã nguồn Online Boutique (repo này, commit `c7cb6d5b`)<br>https://github.com/GoogleCloudPlatform/microservices-demo | Câu log, mã lỗi, timeout, topology, giới hạn tài nguyên. Đường dẫn file và số dòng ghi ngay trong runbook. |
| S13 | Prometheus – Query Functions (`histogram_quantile`)<br>https://prometheus.io/docs/prometheus/latest/querying/functions/ | Cách tính p99 độ trễ từ histogram |
| S14 | Grafana Loki – Log queries / Metric queries<br>https://grafana.com/docs/loki/latest/query/log_queries/<br>https://grafana.com/docs/loki/latest/query/metric_queries/ | Bộ parse `json` đổi dấu `.` thành `_`; `unwrap` và `quantile_over_time` |

## Quy tắc thêm nguồn mới

- Ưu tiên tài liệu chính thức (Kubernetes, Istio, Prometheus, Chaos Mesh, gRPC) và mã nguồn của repo.
- Blog cá nhân hay Stack Overflow chỉ dùng để tham khảo, không dùng làm nguồn chính.
- Ghi ngày kiểm tra. Tài liệu kỹ thuật thay đổi theo phiên bản.
