# PRD — Máy gia công mạch PCB tự động (bản PLC S7-1200)

**Mã đề tài:** 22223DT130  
**Trường:** ĐH Sư phạm Kỹ thuật TP.HCM — Khoa Cơ khí Chế tạo máy, Bộ môn Cơ điện tử  
**GVHD:** ThS. Lê Thanh Tùng  
**SVTH:** Trần Thái An · Trần Ngọc Đức · Nguyễn Văn Lưu  
**Phiên bản thiết kế:** PLC S7-1200 (chuyển đổi từ STM32)

---

## 1. Tổng quan sản phẩm

Máy gia công mạch in PCB tự động theo phương pháp **phay cách ly (isolation milling)**. Thay vì dùng hóa chất ăn mòn, máy dùng dao phay/khoan để bóc lớp đồng dọc theo biên dạng mạch, tạo đường dẫn điện không cần phòng tối hóa học.

Phiên bản này giữ nguyên toàn bộ phần cơ khí từ đồ án gốc; lớp điều khiển được **thiết kế lại hoàn toàn** từ STM32 sang PLC Siemens S7-1200, bổ sung thêm mô-đun kiểm tra quang học tự động (AOI) sau gia công.

### Kiến trúc hệ thống

```
Gerber ──USB──► Pi 4 ──► .nc ──► phân tích, bù z ──Ethernet──► PLC ──► driver ──► động cơ bước
                                                                        │
                       kết quả kiểm tra ◄── YOLO ◄── ảnh ◄── camera ◄────┘
```

- **Raspberry Pi 4 (Master):** đọc USB, chuyển Gerber → `.nc`, phân tích G-code, bù cao độ leveling map, chạy giao diện, điều phối AOI.
- **S7-1200 (Slave):** điều khiển 3 trục bằng PTO, spindle bằng PWM, thay dao tự động, dò leveling, an toàn máy, đóng vai trò bàn định vị camera khi chụp ảnh kiểm tra.

---

## 2. Mục tiêu sản phẩm

| # | Mục tiêu | Chỉ tiêu thành công |
|---|---|---|
| G1 | Gia công đúng thiết kế Gerber | Độ phân giải trục 2,5 µm/xung; sai số hành trình ≤ biên cơ khí |
| G2 | Thay dao tự động tin cậy | ATC không trượt dao trong 100 chu kỳ liên tiếp sau hiệu chỉnh PWM ở 24 V |
| G3 | Phát hiện khuyết tật sau gia công | Recall ≥ 0,95; phân loại được loại khuyết tật |
| G4 | Thời gian kiểm tra chấp nhận được | ≤ 2 phút cho bo 180 × 130 mm |
| G5 | Vận hành an toàn | E-stop cắt cứng nguồn động lực, độc lập PLC |

---

## 3. Người dùng và ngữ cảnh sử dụng

| Nhóm | Mô tả | Kỳ vọng chính |
|---|---|---|
| Sinh viên / kỹ thuật viên PCB | Tự làm bo mạch nguyên mẫu tại phòng lab | Vận hành đơn giản qua UI; không cần kiến thức CNC chuyên sâu |
| Giảng viên / người giám sát | Giám sát quá trình gia công, kiểm tra kết quả | Báo cáo lỗi rõ ràng, bản đồ khuyết tật trực quan |

---

## 4. Thông số máy

| Hạng mục | Giá trị |
|---|---|
| Kích thước máy (D×R×C) | 560 × 490 × 300 mm |
| Hành trình X × Y × Z | 220 × 235 × 50 mm |
| Vùng gia công PCB | 180 × 130 × 2 mm |
| Tốc độ di chuyển | 20 mm/s |
| Tốc độ cắt | 8,33 mm/s |
| Số ổ dao | 6 |
| Điện áp hệ thống | 24 VDC |
| Độ phân giải trục | 2,5 µm/xung (vi bước 1/16) |
| Khối lượng | ~13 kg |

---

## 5. Yêu cầu chức năng

### 5.1 Năm quy trình gia công

| # | Chức năng | Mô tả |
|---|---|---|
| F1 | Phay đường mạch | Phay tách đường đồng theo biên dạng đã offset bán kính dao |
| F2 | Khoan lỗ | Khoan theo tọa độ và đường kính khai báo tường minh trong file Excellon |
| F3 | Phay mặt | Bóc vùng đồng thừa |
| F4 | Cắt viền mạch | Cắt rời bo theo đường viền ngoài |
| F5 | Thay dao tự động (ATC) | 6 ổ dao cố định; spindle đảo chiều để siết/nhả dao |

### 5.2 Kiểm tra quang học tự động (AOI)

| # | Yêu cầu | Chỉ tiêu |
|---|---|---|
| R1 | Phát hiện khuyết tật kích thước ≥ 0,1 mm | Độ phân giải không gian ≤ 0,025 mm/px (16,1 µm/px thực tế) |
| R2 | Tỉ lệ bỏ sót thấp | Recall ≥ 0,95 |
| R3 | Chạy trên phần cứng nhúng | Raspberry Pi 4, không GPU rời |
| R4 | Thời gian kiểm tra | ≤ 2 phút cho bo 180 × 130 mm |
| R5 | Phân loại khuyết tật | Xác định được loại, không chỉ báo có/không |

**Pipeline AOI:**

1. Chụp ghép 16 tile (dùng chuyển động máy làm cơ cấu quét)
2. Ghép panorama + hiệu chỉnh phối cảnh
3. Sinh vùng ứng viên từ tham chiếu Gerber (không dùng golden-template)
4. Phân loại bằng YOLO (ONNX/NCNN) trên từng vùng ứng viên
5. Hiển thị bản đồ lỗi với tọa độ và loại khuyết tật

**8 lớp khuyết tật đặc thù phay cách ly** (phải tự xây dựng tập dữ liệu — không dùng được DeepPCB/HRIPCB vốn dành cho bo ăn mòn).

### 5.3 Hiệu chuẩn và hệ thống

| # | Chức năng | Chi tiết |
|---|---|---|
| S1 | Set home | `MC_Home` active homing; thứ tự Z → X → Y để dao không va phôi |
| S2 | Leveling | Hạ Z bằng `MC_MoveVelocity`; chạm probe → ngắt phần cứng → `MC_Halt` → chốt Z |
| S3 | Leveling map | Ma trận 66 điểm (6×11); ghép mặt phẳng 3 điểm; chạy trên Pi, không phải PLC |
| S4 | Move thủ công | `MC_MoveJog` từng trục từ màn hình |
| S5 | Disable/Enable stepper | `MC_Power` — ngắt để đẩy bàn tay |
| S6 | Hiển thị realtime | PWM spindle, tọa độ x/y/z, chế độ vi bước |

### 5.4 Phần mềm PC (7 tab)

| Tab | Chức năng |
|---|---|
| Menu | Hỏi đã dùng chưa → Yes vào thẳng, No sang video |
| Video Tutorial | Hướng dẫn cho người mới |
| Select File | Nhận file Gerber (lớp đồng, đường viền) + file Excellon (lỗ khoan) |
| Select Area | Chọn vị trí gia công; hiển thị vi bước, đường kính dao, kích thước bo |
| Waiting Convert | Tiến trình xuất file |
| Save File NC | Preview (mô phỏng) hoặc Save file `.nc` |
| End | Home hoặc Exit |

---

## 6. Yêu cầu phi chức năng

| # | Yêu cầu | Chỉ tiêu |
|---|---|---|
| NF1 | An toàn | E-stop cắt cứng nguồn động lực 24 V, không đi qua PLC; tách nguồn PLC khỏi nguồn động lực |
| NF2 | Độ tin cậy | Tuổi thọ thiết kế 5000 giờ |
| NF3 | Nhiễu điện | Cáp xoắn đôi có lưới chắn cho PUL/DIR; opto cách ly sẵn trên driver |
| NF4 | Nhiệt | Tản nhiệt + quạt bắt buộc cho Pi 4 (throttle ở 80 °C) trong tủ điện kín |
| NF5 | Giao tiếp Pi–PLC | `python-snap7` qua Ethernet, ISO-TCP cổng 102; bắt buộc Pi 4 có Gigabit Ethernet onboard |
| NF6 | Định dạng file | G-code chuẩn (`.nc`); mở được bằng bất kỳ phần mềm mô phỏng CNC nào |

---

## 7. Kiến trúc phần cứng

### 7.1 Bộ điều khiển

| Thiết bị | Vai trò |
|---|---|
| Raspberry Pi 4 (4 GB) | Master: xử lý file, giao diện, AOI |
| S7-1200 1214C DC/DC/DC | Slave: chuyển động, I/O, ATC, an toàn — **bắt buộc bản DC/DC/DC** |
| SM 1223 DI8/DQ8 | Mở rộng ngõ ra (nhu cầu 12 DO > 10 DO onboard) |
| TFT LCD 3.5" | Giao diện tại máy |

### 7.2 Driver và công suất

| Thiết bị | Thông số quan trọng |
|---|---|
| Leadshine DM542 (ưu tiên) hoặc TB6600 | 200 kHz max; opto cách ly sẵn |
| Điện trở 2 kΩ / 0,25 W × 9 | **Bắt buộc** hạn dòng opto khi 24 V — bỏ qua là cháy opto ngay lần đầu |
| BTS7960 + board opto | Điều khiển spindle DC RS775 |
| Relay trung gian 24 V × 2 | Đảo chiều spindle (ATC) + ngắt leveling khi spindle quay |

### 7.3 Camera và AOI

| Thiết bị | Thông số quan trọng |
|---|---|
| IMX219 ngàm M12 | **Bắt buộc bản ngàm M12** — Pi Camera Module 2 không lấy nét được ở khoảng cách cần |
| Ống kính M12 8 mm | FOV 52,9 mm ở 123 mm — **đo khoảng hở trước khi mua** |
| Đèn vòng LED khuếch tán | **Bắt buộc** — đồng có phản xạ gương, chiếu thẳng tạo điểm chói bão hòa |

### 7.4 Nguồn

| Thiết bị | Công dụng |
|---|---|
| 24 V / 15 A (360 W) | Mạch động lực: driver, spindle |
| Siemens PM1207 24 V / 2,5 A | PLC và I/O — **tách khỏi nhiễu động lực** |
| Bộ chuyển 24 V → 5 V / 5 A | Pi 4 và LCD |

---

## 8. Kiến trúc phần mềm

### 8.1 Phần mềm Pi 4

| Thành phần | Chức năng |
|---|---|
| App xuất file (PC) | Gerber + Excellon → `.nc` (dùng `pcb2gcode`, bỏ pipeline xử lý ảnh cũ) |
| App giao diện (Pi 4) | UI, đọc USB, phân tích `.nc`, bù z, giao tiếp PLC qua `Service_S7Comm` |
| Module AOI (Pi 4) | Chụp ghép, phân đoạn, sinh ứng viên, phân loại YOLO |

### 8.2 Chương trình PLC (TIA Portal)

| Tầng | Khối |
|---|---|
| Unit | `FB_Axis`×3, `FB_Spindle`, `FB_Probe`, `FB_ToolMagazine`, `FB_IO` |
| Int | `FB_LinearMove`, `FB_ToolChange`, `FB_Leveling`, `FB_Homing` |
| Main | `OB1` + `FB_Sequence` (máy trạng thái) + `FB_Comm` |

**Định dạng G-code trung gian:**

```
G00 X.. Y.. Z..    ; di chuyển nhanh
G01 X.. Y.. F..    ; cắt đường thẳng
G02 / G03          ; cung tròn — phải tuyến tính hóa (PLC không nội suy được)
M03 / M05          ; bật / tắt spindle
M06 T..            ; thay dao
```

### 8.3 Giao tiếp Pi ↔ PLC

`python-snap7`, ISO-TCP cổng 102. Hai cấu hình bắt buộc trong TIA Portal (thiếu → kết nối thành công nhưng đọc toàn số 0):
1. CPU → Protection & Security → **bật "Permit access with PUT/GET"**
2. DB trao đổi → Properties → **bỏ tick "Optimized block access"**

---

## 9. Điểm bắt buộc kiểm chứng trước vận hành

| # | Hạng mục | Lý do |
|---|---|---|
| V1 | **PWM siết/nhả dao ở 24 V** | Thông số đồ án đo ở 12 V; điểm khởi đầu: duty ≈ 39% (PWM ~100/255) |
| V2 | **Điện trở hạn dòng 2 kΩ trước opto driver** | Bỏ qua → cháy opto ngay lần cấp điện đầu |
| V3 | **Thời gian suy luận YOLO trên Pi 4** | Chỉ là ước lượng; đo thực tế để xác nhận chu kỳ kiểm tra ≤ 2 phút |
| V4 | **Đo khoảng hở dầm ngang → mặt bàn trước khi mua ống kính** | Tiêu cự 8 mm cần 123 mm; hẹp hơn → 6 mm (92 mm) hoặc 4 mm (62 mm) |

---

## 10. Thay đổi so với đồ án gốc (STM32)

| Hạng mục | Bản gốc (STM32) | Bản mới (S7-1200) |
|---|---|---|
| Bộ điều khiển | STM32F103C8T6 | S7-1200 1214C + Pi 4 |
| Driver bước | DRV8825 (3,3–5 V logic) | DM542/TB6600 (opto cách ly, 24 V) |
| Điện áp hệ thống | 12 VDC | 24 VDC |
| Giao tiếp Pi–PLC | WiFi (Pi Zero 2W không có Ethernet) | Gigabit Ethernet, giao thức S7 |
| Tủ điện | Bo mạch module tự thiết kế | DIN rail, terminal chuẩn công nghiệp |
| Kiểm tra chất lượng | Thủ công bằng mắt + đồng hồ | AOI tự động với YOLO |
| Định dạng file | Acode tự định nghĩa | G-code chuẩn (`.nc`) |
| Pipeline sinh NC | Xử lý ảnh từ file PDF | Gerber + Excellon → vector, không sai số lượng tử |
| Nội suy trục | Bresenham + BLU 16 hướng | Nội suy đường thẳng tỉ lệ vận tốc (PTO không nội suy arc) |

---

## 11. Tài liệu kỹ thuật liên quan

| File | Nội dung |
|---|---|
| [`docs/thiet-bi-va-chuc-nang.md`](docs/thiet-bi-va-chuc-nang.md) | BOM đầy đủ và danh sách chức năng |
| [`docs/chuc-nang-plc.md`](docs/chuc-nang-plc.md) | Đặc tả PLC: phân bổ I/O, khối hàm, Data Block, an toàn, đấu dây |
| [`docs/gerber-sang-nc.md`](docs/gerber-sang-nc.md) | Sinh đường chạy dao: Gerber + Excellon → `.nc` |
| [`docs/kiem-tra-quang-hoc.md`](docs/kiem-tra-quang-hoc.md) | Kiểm tra quang học tự động (AOI) |
| [`docs/luu-do-giai-thuat.md`](docs/luu-do-giai-thuat.md) | Lưu đồ giải thuật toàn hệ thống |
| [`docs/dinh-vi-va-toi-uu-phoi.md`](docs/dinh-vi-va-toi-uu-phoi.md) | Định vị phôi và tối ưu tận dụng phôi thừa |
| [`docs/so-sanh-stm32-vs-s7.md`](docs/so-sanh-stm32-vs-s7.md) | Đối chiếu bản gốc ↔ bản mới |
