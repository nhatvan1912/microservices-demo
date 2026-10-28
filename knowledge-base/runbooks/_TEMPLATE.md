---
id: RB-XXX
title: <Tên lỗi ngắn gọn>
services: [<service bị lỗi gốc>, <service bị ảnh hưởng>]
fault_type: <dependency_down | pod_down | memory | cpu | network>
severity: <high | medium | low>
chaos_experiment: <PodChaos/pod-failure | NetworkChaos/delay | StressChaos/cpu | ...>
status: draft            # draft = chưa kiểm chứng; verified = đã tiêm lỗi và thấy đúng
sources: [S01, S12]
---

# RB-XXX: <Tên lỗi>

## Tóm tắt
<2–3 câu: lỗi là gì, xảy ra ở đâu, người dùng thấy gì.>

## Cách tái hiện (chaos experiment)
<YAML Chaos Mesh hoặc lệnh kubectl.>

## Triệu chứng: Metrics
| Tín hiệu | PromQL | Bình thường → khi lỗi |
|---|---|---|

## Triệu chứng: Logs
<Câu log copy đúng từ code, kèm file:dòng. Truy vấn LogQL.>

## Lan truyền lỗi
<Service nào bị kéo theo, theo topology.md.>

## Phân biệt với lỗi khác
| Dễ nhầm với | Dấu hiệu để tách |
|---|---|

## Các bước xác minh
<Lệnh kubectl / truy vấn để khẳng định nguyên nhân.>

## Khắc phục
<Cách xử lý ngay và cách phòng tránh lâu dài.>

## Nguồn
- [Sxx] ...
