# Topology: service nào gọi service nào

Lấy từ biến môi trường `*_ADDR` trong `kubernetes-manifests/*.yaml` [S12].
Tất cả kết nối giữa các service là **gRPC**, trừ người dùng → frontend (HTTP)
và cartservice → redis-cart (TCP, giao thức Redis).

```
loadgenerator ──HTTP──► frontend
                          ├──► productcatalogservice
                          ├──► currencyservice
                          ├──► cartservice ──TCP──► redis-cart
                          ├──► recommendationservice ──► productcatalogservice
                          ├──► shippingservice
                          ├──► adservice            (timeout 100ms)
                          ├──► shoppingassistantservice (tùy chọn)
                          └──► checkoutservice
                                  ├──► productcatalogservice
                                  ├──► shippingservice
                                  ├──► paymentservice
                                  ├──► emailservice
                                  ├──► currencyservice
                                  └──► cartservice
```

## Mức độ ảnh hưởng khi một service hỏng

Đọc từ code xử lý lỗi của frontend và checkout [S12]. Đây là thông tin quan
trọng cho RCA: có service hỏng làm sập trang, có service hỏng mà người dùng
gần như không thấy.

| Service hỏng | Ảnh hưởng | Bằng chứng trong code |
|---|---|---|
| productcatalogservice | **Nặng.** Trang chủ, trang sản phẩm, checkout đều lỗi 500 | `handlers.go:69` "could not retrieve products" |
| currencyservice | **Nặng.** Trang chủ lỗi 500 (cần đổi tiền cho mọi sản phẩm) | `handlers.go:64` "could not retrieve currencies", `handlers.go:86` |
| cartservice / redis-cart | **Nặng.** Trang chủ cũng lỗi 500 vì trang chủ đọc giỏ hàng | `handlers.go:74` "could not retrieve cart" |
| checkoutservice | **Vừa.** Chỉ hỏng bước đặt hàng | `placeOrderHandler` (`handlers.go:320`) |
| paymentservice | **Vừa.** Đặt hàng lỗi | `checkoutservice/main.go:254` "failed to charge card" |
| shippingservice | **Vừa.** Xem giỏ hàng và đặt hàng lỗi | `checkoutservice/main.go:260` "shipping error" |
| recommendationservice | **Nhẹ.** Trang vẫn chạy, mất phần gợi ý | `handlers.go:180` Warn "failed to get product recommendations" |
| adservice | **Nhẹ.** Trang vẫn chạy, mất quảng cáo | `handlers.go:530` Warn "failed to retrieve ads" |
| emailservice | **Nhẹ.** Đặt hàng vẫn thành công, không gửi mail | `checkoutservice/main.go:274` Warn "failed to send order confirmation" |

Ghi chú: các đường dẫn `handlers.go`, `rpc.go` nằm trong `src/frontend/`.
