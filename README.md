# Máy gia công mạch PCB tự động — Tài liệu thiết kế (bản PLC S7-1200)

Repo lưu tài liệu thiết kế máy phay/khoan mạch PCB tự động, **chuyển đổi từ kiến trúc STM32 sang
PLC Siemens S7-1200**, bổ sung **kiểm tra quang học tự động sau gia công**.

## Kiến trúc

**Raspberry Pi 4 — Master.** Đọc file Gerber từ USB, chuyển thành `.nc`, phân tích G-code, bù cao độ
theo leveling map, và chạy giao diện. Sau gia công, đảm nhiệm toàn bộ phần thị giác máy tính.

**S7-1200 — Slave.** Điều khiển 3 trục bằng PTO, spindle bằng PWM, thay dao tự động, dò leveling và
an toàn máy. Cũng đóng vai trò bàn định vị cho camera khi chụp ảnh kiểm tra.

Hai bên nối qua Ethernet, giao thức S7 cổng 102 bằng `python-snap7`.

```
Gerber ──USB──► Pi 4 ──► .nc ──► phân tích, bù z ──Ethernet──► PLC ──► driver ──► động cơ bước
                                                                        │
                       kết quả kiểm tra ◄── YOLO ◄── ảnh ◄── camera ◄────┘
```

Phần cơ khí giữ nguyên hoàn toàn từ đồ án gốc; lớp điều khiển được thiết kế lại.

## Tài liệu

| File | Nội dung |
|---|---|
| [`docs/thiet-bi-va-chuc-nang.md`](docs/thiet-bi-va-chuc-nang.md) | Danh sách thiết bị (BOM) và danh sách chức năng đầy đủ |
| [`docs/chuc-nang-plc.md`](docs/chuc-nang-plc.md) | Đặc tả PLC: phân bổ I/O, khối hàm, Data Block, an toàn, đấu dây |
| [`docs/gerber-sang-nc.md`](docs/gerber-sang-nc.md) | Sinh đường chạy dao: Gerber + Excellon → `.nc` |
| [`docs/kiem-tra-quang-hoc.md`](docs/kiem-tra-quang-hoc.md) | Kiểm tra quang học tự động (AOI) bằng thị giác máy và YOLO |
| [`docs/luu-do-giai-thuat.md`](docs/luu-do-giai-thuat.md) | Tập hợp lưu đồ giải thuật toàn hệ thống |
| [`docs/dinh-vi-va-toi-uu-phoi.md`](docs/dinh-vi-va-toi-uu-phoi.md) | Định vị phôi và tối ưu tận dụng phôi thừa |
| [`docs/so-sanh-stm32-vs-s7.md`](docs/so-sanh-stm32-vs-s7.md) | Đối chiếu bản gốc ↔ bản mới: cái mất, cái được |
| [`docs/images/`](docs/images/) | **14 lưu đồ xuất sẵn dạng PNG và SVG** để chèn vào báo cáo |

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

Bốn điểm bắt buộc kiểm chứng trước khi vận hành:

1. **Thông số PWM siết/nhả dao đo ở 12 V nên không còn đúng ở 24 V** — phải thực nghiệm lại
2. **Điện trở hạn dòng 2 kΩ** cho ngõ vào driver ở mức 24 V — thiếu là cháy opto ngay lần cấp điện đầu
3. **Thời gian suy luận YOLO trên Pi 4** mới là ước lượng — phải đo thực tế để xác nhận chu kỳ kiểm tra
4. **Đo khoảng hở từ dầm ngang xuống mặt bàn trước khi mua ống kính** — tiêu cự 12 mm cần 115 mm;
   hẹp hơn dùng 8 mm (77 mm), rộng hơn dùng 16 mm (153 mm)
