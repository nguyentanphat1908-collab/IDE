# Máy gia công mạch PCB tự động — Tài liệu thiết kế (bản PLC S7-1200)

> Repo lưu tài liệu thiết kế máy phay/khoan mạch PCB tự động, **chuyển đổi từ kiến trúc STM32 sang
> PLC Siemens S7-1200**, bổ sung **kiểm tra quang học tự động (AOI) sau gia công**.

---

## Mục lục

1. [Tổng quan hệ thống](#tổng-quan-hệ-thống)
2. [Thông số máy](#thông-số-máy)
3. [Kiến trúc hệ thống](#kiến-trúc-hệ-thống)
4. [Yêu cầu phần cứng](#yêu-cầu-phần-cứng)
5. [Yêu cầu phần mềm](#yêu-cầu-phần-mềm)
6. [Quy trình vận hành](#quy-trình-vận-hành)
7. [Cấu trúc tài liệu](#cấu-trúc-tài-liệu)
8. [Nguồn gốc / Origin](#nguồn-gốc--origin)
9. [Warnings / Lưu ý quan trọng](#warnings--lưu-ý-quan-trọng)

---

## Tổng quan hệ thống

### Mô tả hệ thống

Máy gia công mạch PCB tự động theo phương pháp **phay cách ly (isolation milling)** — tạo đường
mạch bằng cách bóc lớp đồng dọc theo biên dạng thay vì ăn mòn hóa học, loại bỏ hoàn toàn hóa chất
độc hại. Toàn bộ quy trình từ file Gerber đến kiểm tra thành phẩm đều tự động:

```
Gerber ──USB──► Pi 4 ──► .nc ──► phân tích, bù z ──Ethernet──► PLC ──► driver ──► động cơ bước
                                                                        │
                       kết quả kiểm tra ◄── YOLO ◄── ảnh ◄── camera ◄────┘
```

### So sánh STM32 ↔ S7-1200

| Hạng mục | Bản gốc (STM32) | Bản này (S7-1200) |
|---|---|---|
| Bộ điều khiển chính | STM32 | Siemens S7-1200 |
| Điều khiển trục | Bresenham + BLU_mapping | PTO + nội suy vận tốc tỉ lệ |
| Set home | Homing tự triển khai | `MC_Home` Mode 3 |
| Thay dao | Chưa tự động | ATC 6 ổ tự động |
| Kiểm tra chất lượng | Thủ công | AOI tích hợp (camera + YOLO) |
| Điện áp hệ thống | 12 VDC | 24 VDC |

---

## Thông số máy

| Hạng mục | Giá trị | Nguồn |
|---|---|---|
| Kích thước máy (D×R×C) | 560 × 490 × 300 mm | `[ĐA]` |
| Hành trình (X×Y×Z) | 220 × 235 × 50 mm | `[ĐA]` |
| Vùng gia công PCB | 180 × 130 × 2 mm | `[ĐA]` |
| Khối lượng | ~13 kg | `[ĐA]` |
| Tốc độ di chuyển | 20 mm/s | `[ĐA]` |
| Tốc độ cắt | 8,33 mm/s | `[ĐA]` |
| Số ổ dao | 6 | `[ĐA]` |
| Tuổi thọ thiết kế | 5 000 giờ | `[ĐA]` |
| Điện áp hệ thống | **24 VDC** | `[ĐX]` |
| Độ phân giải trục | **2,5 µm/xung** | `[ĐX]` |

Ký hiệu nguồn: `[ĐA]` = đồ án gốc · `[DS]` = datasheet · `[ĐX]` = đề xuất mới, chưa kiểm chứng.

---

## Kiến trúc hệ thống

### Raspberry Pi 4 — Master

- Đọc file Gerber và Excellon từ USB, dùng `pcb2gcode` sinh file `.nc`
- Phân tích G-code, tuyến tính hóa cung tròn, bù cao độ theo leveling map
- Giao diện người dùng trên LCD cảm ứng
- Sau gia công: đảm nhiệm toàn bộ phần thị giác máy tính (AOI)
  - Chụp ảnh 16 tile, phân đoạn HSV, so sánh XOR với mặt nạ Gerber
  - Phân loại khuyết tật bằng YOLOv8 (chạy trực tiếp trên Pi 4, không GPU rời)

### S7-1200 — Slave

- Điều khiển 3 trục (X/Y/Z) bằng **PTO** qua khối `MC_MoveAbsolute`
- Điều khiển spindle bằng **PWM**
- Thực hiện chuỗi ATC (Automatic Tool Change) 6 vị trí
- Dò leveling map 66 điểm bằng **ngắt phần cứng** (độ trễ ~µs)
- Giám sát an toàn: limit switch, alarm driver, watchdog
- Đóng vai trò bàn định vị cho camera khi chụp ảnh kiểm tra

### Giao tiếp

Hai bên nối qua **Ethernet**, giao thức **S7 cổng 102** sử dụng thư viện `python-snap7`.
Ring buffer 32 ô trong `DB_Buffer` giải quyết nghẽn cổ chai khi truyền từng lệnh.

---

## Yêu cầu phần cứng

### BOM

| Thiết bị | SL | Vai trò |
|---|---|---|
| Raspberry Pi 4 (4 GB RAM) | 1 | Master — xử lý Gerber, giao diện, AOI |
| Siemens S7-1214C DC/DC/DC | 1 | PLC — điều khiển trục, spindle, ATC |
| Driver Leadshine DM542 | 3 | Driver động cơ bước 3 trục |
| Động cơ bước NEMA 17 | 3 | Truyền động X/Y/Z |
| Spindle 300 W 12 000 rpm | 1 | Trục chính cắt gọt |
| Camera USB (ống kính C-mount) | 1 | Thu ảnh AOI |
| LCD cảm ứng 7″ | 1 | Giao diện người dùng tại máy |
| Bộ dao (phay đường, khoan, cắt viền) | 6 | Gia công 4 quy trình |

> **Tiêu cự ống kính** phụ thuộc khoảng cách dầm ngang → mặt bàn:
> 8 mm (123 mm) · 6 mm (92 mm) · 4 mm (62 mm) · 12 mm (185 mm).
> **Đo khoảng hở trước khi mua.**

---

## Yêu cầu phần mềm

### Raspberry Pi 4

| Thư viện / Công cụ | Mục đích |
|---|---|
| `pcb2gcode` | Chuyển Gerber + Excellon → file `.nc` |
| `python-snap7` | Giao tiếp S7 qua Ethernet cổng 102 |
| `OpenCV` | Xử lý ảnh, phân đoạn HSV |
| `YOLOv8` (Ultralytics) | Phân loại khuyết tật AOI (D1–D8) |
| `numpy`, `scipy` | Tính toán leveling map, tối ưu vị trí phôi |

### TIA Portal

Lập trình trong **TIA Portal V17+**. Cần cài đặt thêm:

| Gói | Mục đích |
|---|---|
| Motion Control (PLCopen MC) | Khối `MC_Home`, `MC_MoveAbsolute`, `MC_MoveVelocity` |
| GSDML driver HMI | Kết nối màn hình Siemens HMI (nếu dùng) |

---

## Quy trình vận hành

### 9 bước

```
1. Cắm USB chứa file Gerber + Excellon
2. Chọn file và vùng gia công trên LCD
3. Set home: Z → X → Y  ⚠ thứ tự bắt buộc — xem §3 lưu đồ
4. Định vị phôi bằng camera (16 tile quét nhanh, binning 4×)
5. Leveling map 66 điểm (ngắt phần cứng, độ trễ ~µs)
6. Gia công: Pi phân tích G-code → truyền xuống PLC qua ring buffer
7. PLC thực thi, tự động thay dao khi cần (ATC)
8. AOI sau gia công: 16 tile → YOLO → bản đồ khuyết tật D1–D8
9. Hiển thị kết quả trên LCD
```

### State Machine

```
         ┌──────────────────────────────────────────────────────────┐
         │                                                          ▼
IDLE → HOMING → LOCATING → LEVELING → READY → RUNNING → INSPECTING → READY
                                               ↕ PAUSED
              (bất kỳ trạng thái) ──────────► ERROR ──► IDLE (Reset)
```

| Trạng thái | Mô tả |
|---|---|
| `IDLE` | Khởi động, chờ lệnh |
| `HOMING` | Set home Z → X → Y |
| `LOCATING` | Định vị phôi bằng camera |
| `LEVELING` | Dò 66 điểm leveling map |
| `READY` | Sẵn sàng nhận lệnh gia công |
| `RUNNING` | Đang gia công |
| `PAUSED` | Tạm dừng (Stop) |
| `INSPECTING` | AOI sau gia công |
| `ERROR` | Lỗi — chờ Reset |

---

## Cấu trúc tài liệu

| File | Nội dung |
|---|---|
| [`docs/thiet-bi-va-chuc-nang.md`](docs/thiet-bi-va-chuc-nang.md) | Danh sách thiết bị (BOM) đầy đủ và danh sách chức năng |
| [`docs/chuc-nang-plc.md`](docs/chuc-nang-plc.md) | Đặc tả PLC: phân bổ I/O, khối hàm, Data Block, an toàn, đấu dây |
| [`docs/gerber-sang-nc.md`](docs/gerber-sang-nc.md) | Sinh đường chạy dao: Gerber + Excellon → `.nc` |
| [`docs/kiem-tra-quang-hoc.md`](docs/kiem-tra-quang-hoc.md) | Kiểm tra quang học tự động (AOI) — thị giác máy tính và YOLO |
| [`docs/luu-do-giai-thuat.md`](docs/luu-do-giai-thuat.md) | Tập hợp 10 lưu đồ giải thuật toàn hệ thống |
| [`docs/dinh-vi-va-toi-uu-phoi.md`](docs/dinh-vi-va-toi-uu-phoi.md) | Định vị phôi và tối ưu tận dụng phôi thừa |
| [`docs/so-sanh-stm32-vs-s7.md`](docs/so-sanh-stm32-vs-s7.md) | Đối chiếu bản gốc ↔ bản mới: cái mất, cái được |
| [`docs/images/`](docs/images/) | **14 lưu đồ** xuất sẵn dạng PNG và SVG để chèn vào báo cáo |

---

## Nguồn gốc / Origin

Tài liệu dựa trên đồ án tốt nghiệp *"Nghiên cứu, Thiết kế và Chế tạo Máy Gia Công Mạch PCB Tự Động"*
(mã đề tài **22223DT130**), Trường ĐH Sư phạm Kỹ thuật TP.HCM, Khoa Cơ khí Chế tạo máy — Bộ môn Cơ điện tử,
tháng 07/2023.

| | |
|---|---|
| GVHD | ThS. Lê Thanh Tùng |
| SVTH | Trần Thái An (19146304) · Trần Ngọc Đức (19146324) · Nguyễn Văn Lưu (19146355) |
| Thời gian | 03/2023 – 07/2023 (16 tuần) |

Phần cơ khí giữ nguyên hoàn toàn từ đồ án gốc; lớp điều khiển được thiết kế lại.

---

## Warnings / Lưu ý quan trọng

### Quy ước đánh dấu nguồn tin

| Ký hiệu | Ý nghĩa |
|---|---|
| `[ĐA]` | Số liệu trích từ đồ án gốc — đã được tính toán/thực nghiệm |
| `[DS]` | Thông số datasheet nhà sản xuất |
| `[ĐX]` | **Đề xuất thiết kế mới — ước lượng, chưa kiểm chứng, phải đo lại** |

### Bốn điểm bắt buộc kiểm chứng trước khi vận hành

> **Không bỏ qua bốn điểm này — một số có thể gây hỏng phần cứng ngay lần cấp điện đầu.**

1. **Thông số PWM siết/nhả dao đo ở 12 V nên không còn đúng ở 24 V** — phải thực nghiệm lại
   (`T_nhả` và `T_siết` trong `FB_ToolChange`)
2. **Điện trở hạn dòng 2 kΩ** cho ngõ vào driver ở mức 24 V — thiếu là cháy opto ngay lần cấp điện đầu
3. **Thời gian suy luận YOLO trên Pi 4** là ước lượng — phải đo thực tế để xác nhận chu kỳ AOI
   đạt yêu cầu ≤ 2 phút cho bo 180 × 130 mm
4. **Đo khoảng hở từ dầm ngang xuống mặt bàn trước khi mua ống kính** — tiêu cự 8 mm cần 123 mm;
   hẹp hơn dùng 6 mm (92 mm) hoặc 4 mm (62 mm), rộng hơn dùng 12 mm (185 mm)
