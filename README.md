# Máy gia công mạch PCB tự động — Tài liệu thiết kế (bản PLC S7-1200)

Repo lưu tài liệu thiết kế máy phay/khoan mạch PCB tự động, **chuyển đổi từ kiến trúc STM32 sang
PLC Siemens S7-1200**.

**Kiến trúc:** Raspberry Pi 4 làm Master — đọc USB, rasterize PDF, xử lý ảnh, biên dịch Acode và
chạy giao diện. S7-1200 làm Slave — điều khiển 3 trục bằng PTO, spindle bằng PWM, thay dao tự động
và an toàn máy. Hai bên nối qua Ethernet, giao thức S7 trên cổng 102 bằng `python-snap7`.

Phần cơ khí giữ nguyên từ đồ án gốc; toàn bộ lớp điều khiển được thiết kế lại.

## Tài liệu

| File | Nội dung |
|---|---|
| [`docs/thiet-bi-va-chuc-nang.md`](docs/thiet-bi-va-chuc-nang.md) | Danh sách thiết bị (BOM) và danh sách chức năng đầy đủ |
| [`docs/chuc-nang-plc.md`](docs/chuc-nang-plc.md) | Đặc tả PLC: phân bổ I/O, khối hàm, Data Block, an toàn, đấu dây |
| [`docs/xu-ly-anh.md`](docs/xu-ly-anh.md) | Pipeline 12 bước: PDF → Acode polyline |
| [`docs/so-sanh-stm32-vs-s7.md`](docs/so-sanh-stm32-vs-s7.md) | Đối chiếu bản gốc ↔ bản PLC: cái mất, cái được |

## Nguồn gốc

Tài liệu dựa trên đồ án tốt nghiệp *"Nghiên cứu, Thiết kế và Chế tạo Máy Gia Công Mạch PCB Tự Động"*
(mã đề tài 22223DT130), Trường ĐH Sư phạm Kỹ thuật TP.HCM, Khoa Cơ khí Chế tạo máy — Bộ môn Cơ điện tử,
tháng 07/2023. GVHD: ThS. Lê Thanh Tùng. SVTH: Trần Thái An, Trần Ngọc Đức, Nguyễn Văn Lưu.

## Lưu ý khi đọc

Tài liệu đánh dấu rõ ba loại thông tin, **không được lẫn lộn khi sử dụng**:

| Ký hiệu | Ý nghĩa |
|---|---|
| `[ĐA]` | Số liệu trích từ đồ án gốc — đã được tính toán/thực nghiệm |
| `[DS]` | Thông số datasheet nhà sản xuất |
| `[ĐX]` | **Đề xuất thiết kế mới — ước lượng, chưa kiểm chứng, phải đo lại** |

Đặc biệt: **thông số PWM siết/nhả dao của đồ án gốc đo ở 12 V nên không còn đúng khi chuyển sang
24 V** — bắt buộc thực nghiệm lại trước khi vận hành.
