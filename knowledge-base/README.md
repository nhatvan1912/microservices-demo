# Knowledge Base (Runbook) cho hệ thống LLM RCA

Thư mục này chứa "sổ tay bệnh học" cho hệ thống Online Boutique. Khi có alert,
pipeline RAG tìm runbook phù hợp nhất rồi đưa cho LLM đọc, để LLM chẩn đoán
nguyên nhân gốc (root cause).

## Cấu trúc

```
knowledge-base/
├── README.md            ← file này: quy ước viết và cách dùng cho RAG
├── topology.md          ← sơ đồ service nào gọi service nào
├── sources.md           ← danh sách nguồn tham khảo (S01, S02, ...)
└── runbooks/
    ├── _TEMPLATE.md     ← mẫu để viết runbook mới
    ├── RB-001-redis-cart-unavailable.md
    ├── RB-002-service-pod-down.md
    ├── RB-003-oomkilled-memory.md
    ├── RB-004-cpu-saturation.md
    └── RB-005-network-latency.md
```

## Điều kiện để runbook dùng được

Runbook giả định cluster đã có:

| Thành phần | Cung cấp metric | Ghi chú |
|---|---|---|
| Istio (sidecar injection bật cho namespace `default`) | `istio_requests_total`, `istio_request_duration_milliseconds`, `istio_tcp_*` | Các service của Boutique **không** tự xuất metric Prometheus. Không có Istio thì không có độ trễ và tỉ lệ lỗi theo từng service. |
| kube-state-metrics | `kube_pod_container_status_*` | Trạng thái pod, số lần restart, lý do bị kill |
| cAdvisor (có sẵn trong kubelet) | `container_cpu_*`, `container_memory_*` | CPU, RAM của từng container |
| Loki + Promtail | Log | Tên label (`app`, `container`, ...) phụ thuộc cấu hình Promtail, cần chỉnh lại truy vấn LogQL cho khớp |
| Chaos Mesh | — | Dùng để tiêm lỗi. Mỗi runbook có mục "Cách tái hiện". |

## Quy ước viết runbook

1. **Mỗi runbook = một kiểu lỗi.** Không gộp nhiều lỗi vào một file.
2. **Phần front-matter (YAML ở đầu file) bắt buộc.** RAG dùng nó để lọc theo `services` và `fault_type`.
3. **Viết tiếng Việt, giữ nguyên tiếng Anh cho từ khóa kỹ thuật**: tên metric, câu log, mã lỗi gRPC, tên service. Lý do: alert và log đều là tiếng Anh, giữ nguyên thì tìm kiếm theo từ khóa (BM25) mới khớp.
4. **Câu log phải copy đúng từ code**, kèm đường dẫn file và số dòng. Không tự bịa câu log.
5. **Mỗi khẳng định kỹ thuật phải có nguồn**, ghi dạng `[S01]` trỏ tới [sources.md](sources.md).
6. **Phải có mục "Phân biệt với lỗi khác".** Đây là phần quan trọng nhất cho RCA: nhiều lỗi có triệu chứng giống nhau, LLM cần biết dấu hiệu nào giúp tách chúng ra.
7. **Trường `status`:**
   - `draft`: viết từ tài liệu và đọc code, chưa tiêm lỗi thật để kiểm chứng.
   - `verified`: đã tiêm lỗi, đã thấy đúng các triệu chứng, đã chụp lại số liệu.

   Hiện tại **tất cả runbook đều là `draft`**. Sau khi Quang dựng xong hệ thống, cần chạy từng thí nghiệm chaos, sửa lại số liệu thực tế và chuyển sang `verified`.

## Cách dùng cho RAG (đề xuất)

```
Alert (số liệu bất thường)
   │
   ▼
1. Đổi alert thành câu mô tả (text), KHÔNG embedding số thô
   ví dụ: "frontend lỗi 5xx tăng; cartservice trả grpc status 9;
           log frontend có 'Can't access cart storage'"
   │
   ▼
2. Tìm runbook: kết hợp
   - Vector search (embedding câu mô tả)
   - Keyword search BM25 (khớp tên metric, câu log, mã lỗi)
   - Lọc theo metadata `services`
   │
   ▼
3. Lấy top 3 runbook → ghép vào prompt cùng alert, metrics, logs, topology
   │
   ▼
4. LLM trả lời: nguyên nhân gốc, bằng chứng, runbook đã dùng
```

**Chia nhỏ (chunking):** cắt theo heading `##` của runbook. Mỗi chunk mang theo front-matter của runbook gốc để khi tìm thấy một đoạn vẫn biết nó thuộc runbook nào.

**Đánh giá:** làm một bộ khoảng 20 alert mẫu, mỗi alert có sẵn runbook đúng. Đo tỉ lệ runbook đúng nằm trong top 3 (Recall@3).
