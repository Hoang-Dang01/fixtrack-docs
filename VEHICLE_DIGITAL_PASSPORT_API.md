# Vehicle Digital Passport — Backend API Contract

## Mục tiêu

Mỗi phương tiện có một QR token ngẫu nhiên. QR chỉ chứa URL/token định danh, không nhúng số khung, số máy hoặc lịch sử sửa chữa. Khi quét, client gọi backend để nhận dữ liệu mới nhất theo quyền của người dùng.

## Phân lớp dữ liệu

### Tra cứu công khai

`GET /tra-cuu/{qr_token}` không yêu cầu đăng nhập và chỉ trả:

- Mã, tên, loại, hãng/model và trạng thái xe.
- Năm sản xuất và tuổi xe tính theo năm.
- Khu vực quản lý.
- Tổng số lần sửa chữa và thời điểm sửa gần nhất.

API công khai không trả số khung, số máy, mô tả lỗi chi tiết, vật tư hoặc thông tin nhân sự.

### Hồ sơ nội bộ

`GET /vehicles/{vehicle_id}/passport` yêu cầu JWT và trả:

- Hồ sơ kỹ thuật gồm số khung, số máy và ngày đưa vào sử dụng.
- Tuổi xe được backend tính tại thời điểm truy vấn.
- QR token hiện hành.
- Tổng số lần sửa và tổng số ngày nằm xưởng.
- Timeline phiếu sửa chữa, lỗi và vật tư đã cấp.

Admin, cơ giới và kỹ thuật được xem hồ sơ nội bộ. Tài xế chỉ được xem phương tiện đang được phân công; vai trò khác mặc định bị từ chối.

## Quản trị QR

- `POST /qr-codes`: admin tạo QR token ngẫu nhiên cho xe chưa có QR.
- `POST /qr-codes/{vehicle_id}/regenerate`: admin thu hồi token cũ và cấp token mới.
- Sau khi regenerate, token cũ trả HTTP 404.

QR nên mã hóa URL frontend dạng:

```text
https://<frontend-host>/tra-cuu/<qr_token>
```

Frontend đọc token từ URL rồi gọi `GET /tra-cuu/{qr_token}`. Nếu người dùng đăng nhập và có quyền, frontend có thể chuyển sang internal passport.

## Dữ liệu phương tiện bổ sung

Các payload tạo/cập nhật/đọc phương tiện hỗ trợ:

- `hang_san_xuat`
- `model`
- `so_khung` — unique nếu có
- `so_may` — unique nếu có
- `nam_san_xuat` — 1900 đến 2100
- `ngay_dua_vao_su_dung`

## Migration và tương thích

Migration `20260711_0003` thêm các trường hồ sơ kỹ thuật, unique index và rotate toàn bộ QR token kế thừa dễ đoán. Client không được lưu cứng token cũ. Sau migration cần in lại QR nếu QR vật lý đang chứa URL/token cũ.

## Kết quả kiểm thử

- QR mới không chứa mã phương tiện.
- Public view không lộ số khung/số máy.
- Regenerate làm token cũ mất hiệu lực.
- Tài xế bị từ chối trước khi được giao xe và được phép sau khi có assignment.
- Toàn bộ regression suite: `29 passed`.
