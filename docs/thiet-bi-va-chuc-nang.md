# Máy gia công mạch PCB tự động — Danh sách thiết bị & chức năng

> **Bản thiết kế lại trên nền PLC Siemens S7-1200.**
> Đây **không phải** cấu hình gốc của đồ án. Phần cơ khí giữ nguyên, toàn bộ lớp điều khiển
> được thay mới. Xem `so-sanh-stm32-vs-s7.md` để đối chiếu bản gốc và bản này.

## Nguồn gốc

| | |
|---|---|
| Đồ án | Nghiên cứu, Thiết kế và Chế tạo Máy Gia Công Mạch PCB Tự Động |
| Trường | ĐH Sư phạm Kỹ thuật TP.HCM — Khoa Cơ khí Chế tạo máy, Bộ môn Cơ điện tử |
| Mã đề tài | 22223DT130 |
| GVHD | ThS. Lê Thanh Tùng |
| SVTH | Trần Thái An (19146304) · Trần Ngọc Đức (19146324) · Nguyễn Văn Lưu (19146355) |
| Thời gian | 03/2023 – 07/2023 (16 tuần) |

## Quy ước đánh dấu nguồn tin

Ba loại thông tin trong tài liệu này **không được lẫn lộn**:

| Ký hiệu | Ý nghĩa |
|---|---|
| `[ĐA]` | Số liệu trích trực tiếp từ đồ án gốc — đã được nhóm tác giả tính toán/thực nghiệm |
| `[DS]` | Thông số datasheet của nhà sản xuất (Siemens, Leadshine, Raspberry Pi…) |
| `[ĐX]` | **Đề xuất thiết kế mới — ước lượng, chưa kiểm chứng thực tế, phải đo lại** |

## Thông số máy

| Hạng mục | Giá trị | |
|---|---|---|
| Kích thước máy (D×R×C) | 560 × 490 × 300 mm | `[ĐA]` |
| Hành trình máy (X×Y×Z) | 220 × 235 × 50 mm | `[ĐA]` |
| Vùng gia công PCB | 180 × 130 × 2 mm | `[ĐA]` |
| Khối lượng | ~13 kg | `[ĐA]` |
| Tốc độ di chuyển | 20 mm/s | `[ĐA]` |
| Tốc độ cắt | 8,33 mm/s | `[ĐA]` |
| Số ổ dao | 6 ổ | `[ĐA]` |
| Tuổi thọ thiết kế | 5000 giờ | `[ĐA]` |
| **Điện áp hệ thống** | **24 VDC** | `[ĐX]` — bản gốc dùng 12 V |
| **Độ phân giải trục** | **2,5 µm/xung** | `[ĐX]` — xem §A7 |

---

# PHẦN A — DANH SÁCH THIẾT BỊ

## A1. Cơ khí — giữ nguyên 100% từ đồ án gốc

Toàn bộ phần cơ khí không đổi khi chuyển sang PLC. Mọi số liệu dưới đây là `[ĐA]`.

| # | Thiết bị | SL | Thông số | Chức năng |
|---|---|---|---|---|
| 1 | Trục vitme T8, thép không gỉ | 3 | d = 8 mm · d₁ = 6 mm · d₂ = 7 mm · bước ren p = 2 mm · 4 mối ren → **bước vít 8 mm** · góc nâng γ = 20°. Chiều dài: X = 394, Y = 498, Z = 130 mm | Biến chuyển động quay của động cơ thành tịnh tiến cho 3 trục |
| 2 | Đai ốc đồng thanh | 3 | H = 14 mm · ~7 vòng ren · hệ số ma sát f = 0,12 · góc ma sát ρ = 7,08° | Ăn khớp vitme, mang bàn máy và cụm trục chính |
| 3 | Thanh trượt tròn + gối đỡ + con trượt | 3 bộ | — | Dẫn hướng X/Y/Z, chịu phần lớn tải trọng thay cho vitme |
| 4 | Gối đỡ vòng bi KFL08 + bi SU08 | — | 48 × 27 × 12 mm · d trong 8 mm · tải động C = 3,27 kN · tải tĩnh C₀ = 1,37 kN | Đỡ hai đầu trục vitme, chịu lực dọc trục |
| 5 | Ổ chứa dao thẳng hàng cố định | 1 (6 ổ) | Gắn cố định trên bàn máy | Chứa dao cho hệ thay dao tự động (ATC) |
| 6 | Khung máy + bàn máy | 1 | Bàn máy di chuyển X–Y, cụm trục chính chỉ di chuyển Z | Định hình, chịu lực cắt |
| 7 | Bộ dao | 6 | Dao phay đường, mũi khoan các cỡ, dao cắt viền | Phục vụ 4 quy trình gia công |

**Cơ sở tính toán từ đồ án** `[ĐA]`: lực dọc trục lớn nhất X = 3,06 N · Y = 9,5 N · **Z = 12 N**.
Kiểm bền: σ_tđ = 1,8 MPa ≪ [σ] = 68,3 MPa. Kiểm ổn định: hệ số an toàn 55–541.
Ổ lăn: C_đ = 0,043 kN ≪ C = 3,27 kN. Tất cả đều dư biên rất lớn.

## A2. Động cơ — giữ nguyên, đổi điện áp vận hành

| # | Thiết bị | SL | Thông số | Chức năng |
|---|---|---|---|---|
| 8 | Động cơ bước **NEMA17** | 3 | 42 × 42 × 40 mm · **12–24 V** · momen 0,45 N.m · dòng 1,6 A · góc bước 1,8° `[ĐA]` | Dẫn động 3 trục X, Y, Z |
| 9 | Động cơ DC **RS775** | 1 | **12–24 V** · 1,2 A không tải · 3,25 A có tải · 7500–12000 v/ph `[ĐA]` | Trục chính: quay dao phay/khoan/cắt **và** tạo momen siết–nhả dao cho ATC |

Cả hai đều nằm trong dải cho phép ở 24 V, nên chuyển điện áp không cần đổi động cơ.

> **Cảnh báo — thông số ATC hết hiệu lực.** `[ĐA]` đo ở **12 V**: siết dao PWM 200 / 200 ms (5/5 lần),
> nhả dao PWM 250 / 250 ms (5/5 lần). Ở 24 V các giá trị này **không còn đúng** và sẽ làm hỏng
> cơ cấu kẹp dao. Điểm khởi đầu để đo lại `[ĐX]`: giữ điện áp hiệu dụng ~9,4 V → duty ≈ 39%
> (PWM ≈ 100/255). **Bắt buộc lặp lại quy trình thực nghiệm của đồ án ở 24 V trước khi vận hành.**

**Yêu cầu tối thiểu từ tính toán đồ án** `[ĐA]`: T ≥ 0,0215 N.m · P ≥ 0,375 W · n ≥ 150 v/ph.
NEMA17 (0,45 N.m) dư rất nhiều biên.

## A3. Điều khiển — viết lại hoàn toàn

### A3.1 Bộ điều khiển

| # | Thiết bị | SL | Thông số | Chức năng |
|---|---|---|---|---|
| 10 | **Raspberry Pi 4 (4 GB)** | 1 | 4× Cortex-A72 · 4 GB LPDDR4 · **Gigabit Ethernet** · 2× USB 3.0 · 40 GPIO · 5 V/3 A `[DS]` | **Master**: đọc USB, sinh `.nc`, biên dịch, chạy giao diện và module AOI |
| 11 | **CPU S7-1200 1214C DC/DC/DC** | 1 | 14 DI / 10 DO / 2 AI · **4 bộ phát xung PTO/PWM, 100 kHz** · 1 cổng PROFINET `[DS]` | **Slave**: điều khiển chuyển động, I/O, ATC, an toàn |
| 12 | **SM 1223 DI8/DQ8 ×24VDC** | 1 | 8 DI / 8 DO transistor `[DS]` | Bù ngõ ra thiếu — nhu cầu 12 DO > 10 DO onboard |
| 13 | Màn hình cảm ứng **TFT LCD 3.5"** | 1 | 320 × 480 px · 50 Hz · 5 V/120 mA `[ĐA]` | Giao diện vận hành tại máy |

> **Bắt buộc bản DC/DC/DC.** Bản DC/DC/**Rly** (ngõ ra relay) **không phát xung được** — chọn nhầm
> là toàn bộ thiết kế chuyển động không chạy.

**Ba lý do nâng Pi Zero 2W → Pi 4** (thay đổi có căn cứ, không phải nâng cho mạnh):

1. **Gigabit Ethernet onboard.** Pi Zero 2W **không có cổng Ethernet nào** — chỉ WiFi. Link snap7
   tới PLC lẽ ra phải qua adapter USB hoặc WiFi: không tất định, dễ nhiễu trong tủ điện.
2. **RAM 4 GB thay 512 MB.** Ảnh kiểm tra quang học sau gia công gồm 20 tile × 11,9 MP ≈ 0,72 GB
   dữ liệu thô — **không chạy nổi trên 512 MB**. Xem `kiem-tra-quang-hoc.md` §5.2.
3. **4× Cortex-A72 + USB 3.0.** Cần cho suy luận YOLO trên thiết bị biên không GPU, và rút ngắn
   thời gian đọc USB.

### A3.2 Driver và mạch công suất

| # | Thiết bị | SL | Thông số | Chức năng |
|---|---|---|---|---|
| 14 | **Leadshine DM542** *(ưu tiên)* hoặc TB6600 | 3 | DM542: 20–50 VDC · 1,0–4,2 A · vi bước tới 1/128 · **tần số xung tới 200 kHz** · opto cách ly sẵn `[DS]` | Cấp năng lượng & điều khiển 3 động cơ bước |
| 15 | **Điện trở 2 kΩ / 0,25 W** | 9 | — | **Hạn dòng opto driver khi điều khiển mức 24 V** (3 trục × PUL/DIR/ENA) |
| 16 | **BTS7960** + board opto 24 V→5 V | 1 | BTS7960: 6–27 V · 43 A max `[ĐA]`. Logic vào **3,3–5 V** `[DS]` | Điều khiển tốc độ và chiều động cơ trục chính |
| 17 | Relay trung gian 24 V | 2 | — | ① Đảo chiều spindle cho ATC · ② Ngắt tín hiệu leveling khi spindle quay |

> **Vì sao ưu tiên DM542 hơn TB6600:** biên tần số 200 kHz so với ~20 kHz, và tài liệu điện trở
> hạn dòng được ghi rõ ràng (5 V không cần · 12 V thêm 1 kΩ · 24 V thêm 2 kΩ). TB6600 trên thị
> trường có nhiều bản clone khác nhau — **phải tra datasheet đúng model mua, không suy đoán**.

> **Bỏ qua điện trở hạn dòng là cháy opto driver ngay lần cấp điện đầu tiên.** Xem chi tiết đấu dây
> ở `chuc-nang-plc.md` §7.

### A3.3 Cảm biến và nút điều khiển

| # | Thiết bị | SL | Chức năng |
|---|---|---|---|
| 18 | Công tắc hành trình (gốc) | 3 | Cam về gốc cho `MC_Home` trục X/Y/Z |
| 19 | Công tắc hành trình (hard limit) | 3 | Chặn cứng cuối hành trình |
| 20 | Cảm biến leveling | 1 | Dò chạm mặt đồng — **đấu vào DI có ngắt phần cứng** |
| 21 | **Nút E-stop + relay an toàn** | 1 | **Đấu cứng cắt nguồn 24 V động lực, không đi qua PLC** |
| 22 | Nút Start / Stop / Reset | 3 | Vận hành cơ bản |
| 23 | Đèn báo (xanh/đỏ) + còi | 3 | Trạng thái chạy / lỗi |

Số lượng công tắc hành trình là `[ĐX]` — đồ án gốc không nêu rõ số lượng.

### A3.4 Nguồn

| # | Thiết bị | SL | Thông số | Chức năng |
|---|---|---|---|---|
| 24 | Nguồn xung **24 V / 15 A (360 W)** | 1 | Vào 110–220 VAC | Cấp mạch động lực: driver bước, spindle |
| 25 | **Siemens PM1207 24 V / 2,5 A** | 1 | 60 W | **Cấp riêng cho PLC và I/O — tách khỏi nhiễu động cơ** |
| 26 | Bộ chuyển **24 V → 5 V / 5 A** | 1 | — | Cấp Raspberry Pi 4 và LCD |

> **Pi 4 cần đủ 5 V/3 A.** Thiếu dòng gây under-voltage throttling và hỏng thẻ SD.
> **Đấu dây:** cấp qua **USB-C** thì có mạch bảo vệ sẵn của Pi. Cấp thẳng vào chân 5 V trên header
> 40 chân thì **bỏ qua toàn bộ bảo vệ** — nếu chọn cách này phải tự thêm cầu chì và đảm bảo bộ
> chuyển không vọt áp lúc đóng nguồn.

> **Vì sao tách hai nguồn:** đây là thực hành chuẩn công nghiệp. Nhiễu từ driver bước và spindle
> nếu chung nguồn với PLC sẽ gây reset CPU hoặc đọc sai ngõ vào.

### A3.5 Vật tư tủ điện và cáp

| # | Vật tư | Ghi chú |
|---|---|---|
| 27 | **Cáp xoắn đôi có lưới chắn** | Cho PUL/DIR và cáp driver→động cơ — **thay cho mạch khử rung tự chế của đồ án gốc** |
| 28 | Cáp mạng Cat5e + đầu RJ45 | Pi 4 ↔ PLC. **Không cần switch** — cổng PROFINET có auto-crossover |
| 29 | Tủ điện, DIN rail, terminal, aptomat, cầu chì | — |
| 30 | **Tản nhiệt + quạt cho Pi 4** | Bắt buộc trong tủ kín cạnh spindle và driver — Pi 4 throttle ở 80 °C `[DS]` |
| 31 | Quạt thông gió tủ điện | Điều khiển từ PLC |

### A3.6 Cụm kiểm tra quang học (AOI) — mới hoàn toàn

| # | Thiết bị | SL | Thông số | Chức năng |
|---|---|---|---|---|
| 32 | **Raspberry Pi Camera Module 3** | 1 | Sony IMX708 · 11,9 MP (4608 × 2592) · lấy nét tự động · giao tiếp CSI `[DS]` | Chụp ảnh bo sau gia công |
| 33 | **Đèn vòng LED khuếch tán** | 1 | CRI cao, nhiệt độ màu cố định | **Triệt phản xạ gương trên bề mặt đồng** |
| 34 | Giá gắn camera + đèn | 1 | Gắn trên cụm trục chính, di chuyển cùng | Chụp ghép theo lưới bằng chính chuyển động máy |
| 35 | Che chắn khoang chụp | 1 | — | Loại nhiễu do ánh sáng phòng thay đổi |
| 36 | *(Tùy chọn)* Google Coral USB Accelerator | 1 | Edge TPU | Tăng tốc suy luận YOLO nếu cần |

> **Chiếu sáng khuếch tán là bắt buộc, không phải tùy chọn.** Đồng có phản xạ gương mạnh; chiếu sáng
> trực tiếp tạo điểm chói bão hòa làm hỏng bước phân đoạn đồng/nền. Xem `kiem-tra-quang-hoc.md` §5.3.

> **Camera gắn trên cụm trục chính** để tận dụng cơ cấu định vị 2,5 µm của máy làm phương tiện quét
> ảnh — đạt độ phân giải 13 µm/px mà không cần ống kính telecentric đắt tiền.

## A4. Phần mềm

| # | Thành phần | Nền tảng | Chức năng | Trạng thái |
|---|---|---|---|---|
| 37 | **App xuất file** (PC) | Python + Tkinter, kiến trúc **Framework_ADL** `[ĐA]` | Gerber + Excellon → file `.nc` | **Viết lại: bỏ xử lý ảnh, dùng pcb2gcode** |
| 38 | **App giao diện** (Pi 4) | Python, Framework_ADL `[ĐA]` | UI, đọc USB, phân tích `.nc`, bù z, giao tiếp PLC | **Thêm `Service_S7Comm` (python-snap7)** |
| 39 | **Module AOI** (Pi 4) | Python + OpenCV + YOLO (ONNX/NCNN) | Chụp ghép, phân đoạn, sinh ứng viên, phát hiện khuyết tật | **Mới hoàn toàn** |
| 40 | **Chương trình PLC** | TIA Portal, SCL + LAD | Chuyển động, I/O, ATC, chụp ảnh, an toàn | **Mới hoàn toàn — thay Firmware_ADL** |

## A5. Phân bổ I/O

Bảng đầy đủ ở **`chuc-nang-plc.md` §2** — không lặp lại ở đây để tránh hai bảng lệch nhau khi sửa.

Tóm tắt: PTO1–3 trên `Q0.0`–`Q0.5`, PWM4 trên `Q0.6` (**bắt buộc onboard**, không đặt được trên SM).
Tổng sử dụng **15/18 DO** và **14/22 DI**.

> **Toàn bộ 10 ngõ ra onboard đã dùng hết.** Còn dư 3 DO (trên SM) và 8 DI — đủ cho module phủ keo
> UV ở mức cơ bản. Nhưng vì ngõ ra SM không phát xung được, **muốn thêm trục hoặc thêm kênh PWM thì
> phải đổi CPU**, không mở rộng bằng module được.

## A6. Cân bằng công suất 24 V

| Phụ tải | Dòng | |
|---|---|---|
| 3 × driver bước | ~6,0 A | `[ĐX]` |
| Động cơ RS775 (có tải) | 3,25 A | `[ĐA]` |
| CPU 1214C + SM 1223 | ~0,5 A | `[ĐX]` |
| Raspberry Pi 4 qua bộ chuyển (15 W ÷ 24 V ÷ 0,9) | ~0,7 A | `[ĐX]` |
| LCD 3.5" quy về phía 24 V | ~0,03 A | `[ĐX]` |
| Relay, đèn, quạt | ~0,5 A | `[ĐX]` |
| **Tổng** | **≈ 11 A → chọn nguồn 15 A** | |

Đối chiếu bản gốc 12 V: tổng 15,57 A `[ĐA]`. Pi 4 tiêu thụ nhiều hơn Zero 2W nhưng quy về phía 24 V
thì chênh lệch không đáng kể — **không đổi kết luận chọn nguồn**.

## A7. Kiểm chứng chọn CPU — tần số xung và độ phân giải

Đây là cơ sở định lượng để khẳng định S7-1200 đủ sức điều khiển máy này:

```
Bước vít           = 8 mm/vòng                    [ĐA]
Tốc độ di chuyển   = 20 mm/s                      [ĐA]
→ Tốc độ quay      = 20 / 8      = 2,5 vòng/s
Góc bước 1,8°      = 200 bước/vòng                [ĐA]
Vi bước 1/16       = 3200 xung/vòng
→ Tần số xung      = 2,5 × 3200  = 8 000 Hz  = 8 kHz
→ Độ phân giải     = 8 / 3200    = 0,0025 mm = 2,5 µm/xung
→ Độ rộng xung     = 62,5 µs (PTO 50% duty tại 8 kHz)
```

| Chỉ tiêu | Yêu cầu | PTO S7-1200 | Biên |
|---|---|---|---|
| Tần số xung | 8 kHz | 100 kHz `[DS]` | **12,5×** |
| Độ rộng xung | 62,5 µs | ≥ 2,5 µs (DM542) `[DS]` | **25×** |

Kể cả nâng lên vi bước 1/32 (16 kHz) vẫn còn dư rất nhiều biên.

## A8. Thiết bị bị loại bỏ so với đồ án gốc

| Thiết bị gốc | Lý do loại |
|---|---|
| **STM32F103C8T6** | Thay bằng S7-1200 |
| **3× DRV8825** | Logic 3,3–5 V không nhận thẳng 24 V từ PLC; thay bằng driver công nghiệp |
| **Mạch chống nhiễu / khử rung tự chế** | DM542/TB6600 đã có opto cách ly; nhiễu xử lý gốc rễ bằng cáp lưới chắn và đi dây đúng chuẩn |
| **Nguồn xung 12 V – 20 A** | Toàn máy chuyển sang 24 V |
| **Bo mạch module tự thiết kế** | Thay bằng tủ điện, DIN rail, terminal chuẩn công nghiệp |
| **Raspberry Pi Zero 2W** | Không có cổng Ethernet; 512 MB không đủ cho ảnh AOI và suy luận YOLO |
| **Pipeline PDF + xử lý ảnh** | Thay bằng Gerber → `.nc`: dữ liệu vector, không sai số lượng tử hóa, các layer đồng bộ theo chuẩn |

---

# PHẦN B — DANH SÁCH CHỨC NĂNG

## B1. Năm quy trình gia công

Người dùng tick chọn quy trình cần chạy ở tab Select mode `[ĐA]`.

| # | Chức năng | Mô tả |
|---|---|---|
| 1 | **Phay đường mạch** | Phay tách đường đồng theo biên dạng đã offset bán kính dao |
| 2 | **Khoan lỗ** | Khoan theo tọa độ và đường kính **khai báo tường minh trong file Excellon** |
| 3 | **Phay mặt** | Bóc vùng đồng thừa |
| 4 | **Cắt viền mạch** | Cắt rời bo theo đường viền ngoài |
| 5 | **Thay dao tự động (ATC)** | 6 ổ dao cố định, spindle đảo chiều để siết/nhả |
| 6 | **Kiểm tra quang học sau gia công (AOI)** | **Mới** — chụp ghép 20 tile, phát hiện và phân loại 8 lớp khuyết tật, hiển thị bản đồ lỗi. Xem `kiem-tra-quang-hoc.md` |

## B2. Chức năng hệ thống và hiệu chuẩn

| # | Chức năng | Hiện thực trên bản PLC |
|---|---|---|
| 7 | **Set home** | `MC_Home` chế độ active homing, dùng công tắc hành trình làm cam. Thứ tự **Z → X → Y** để dao không va phôi |
| 8 | **Leveling** | Hạ Z chậm bằng `MC_MoveVelocity`, chạm probe → **ngắt phần cứng** → `MC_Halt` → chốt Z |
| 9 | **Leveling map** | **Giữ nguyên 66 điểm ma trận 6×11** `[ĐA]`. Ghép mỗi 3 điểm thành 1 mặt phẳng + hàm chọn mặt phẳng phù hợp cho (x,y). **Chạy trên Pi, không phải PLC** |
| 10 | **Move thủ công** | `MC_MoveJog` từng trục từ màn hình |
| 11 | **Disable / Enable stepper** | `MC_Power` — ngắt để đẩy bàn máy bằng tay |
| 12 | **Unit control** | Hiệu chỉnh thông số máy |
| 13 | **Info** | Hiển thị thông tin máy |
| 14 | **Hiển thị realtime** | PWM spindle, thời gian giữa 2 bước xung, chế độ vi bước, tọa độ x/y/z |

## B3. Bảy tab phần mềm PC — giữ nguyên luồng của đồ án gốc

| # | Tab | Chức năng |
|---|---|---|
| 15 | Menu | Hỏi đã dùng máy chưa → Yes vào thẳng, No sang video |
| 16 | Video Tutorial | Hướng dẫn cho người mới |
| 17 | Select File | Nhận **file Gerber** (lớp đồng, đường viền) + **file Excellon** (lỗ khoan) |
| 18 | Select Area | Chọn vị trí gia công trên phôi; hiển thị chế độ vi bước, đường kính dao, kích thước bo |
| 19 | Waiting Convert | Tiến trình xuất file |
| 20 | Save File NC | **Preview** (mô phỏng) hoặc **Save file** — file `.nc` mở được bằng phần mềm mô phỏng CNC bất kỳ |
| 21 | End | Home hoặc Exit |

## B4. Giải thuật lõi

| # | Giải thuật | Trạng thái |
|---|---|---|
| 22 | **Gerber + Excellon → `.nc`** | **Thay hoàn toàn** pipeline xử lý ảnh của đồ án gốc. Chi tiết ở `gerber-sang-nc.md` |
| 23 | **Nội suy đường thẳng 2 trục** | **Mới** — vận tốc tỉ lệ `v_x = v·\|dx\|/L`, `v_y = v·\|dy\|/L` để 2 trục về đích cùng lúc |
| 24 | **Bắt tay truyền dữ liệu** | **Giữ nguyên triết lý** "truyền không nhanh, nhưng phải đủ" `[ĐA]` — cặp đếm `int_sys`/`int_cnc` chuyển thành 2 word trong DB; quá hạn thì gửi lại lệnh cũ |
| 25 | ~~Bresenham Line~~ | **Bị loại bỏ** — PTO không nội suy được |
| 26 | ~~BLU_mapping 16 hướng~~ | **Bị loại bỏ** — cùng lý do |
| 27 | **Kiến trúc AOI lai** | **Mới** — sinh vùng ứng viên bằng tham chiếu Gerber + phân loại bằng YOLO. Chi tiết ở `kiem-tra-quang-hoc.md` |

**Định dạng trung gian: G-code chuẩn thay cho Acode tự định nghĩa**

```
G00 X.. Y.. Z..    ; di chuyển nhanh, dao nâng
G01 X.. Y.. F..    ; cắt theo đường thẳng
G02 / G03          ; cung tròn — PHẢI tuyến tính hóa, PLC không nội suy được
M03 / M05          ; bật / tắt trục chính
M06 T..            ; thay dao
```

> **Vì sao bỏ Acode.** File `.nc` là G-code chuẩn nên **mở được bằng bất kỳ phần mềm mô phỏng CNC
> nào** để kiểm tra đường chạy dao trước khi gia công — điều mà định dạng tự định nghĩa không làm
> được. Đổi lại, mất đi tính "tự thiết kế định dạng" vốn là một đóng góp của đồ án gốc.

> **Ràng buộc cung tròn.** `G02`/`G03` phải được tuyến tính hóa vì các trục PTO của S7-1200 chạy
> độc lập, chỉ dựng được đường thẳng. Xem `gerber-sang-nc.md` §6.

## B5. Chức năng phía PLC

Chi tiết đầy đủ ở **`chuc-nang-plc.md`**. Tổng quan ánh xạ kiến trúc:

| Firmware_ADL (đồ án gốc) | TIA Portal (bản mới) |
|---|---|
| Tầng **Unit** | `FB_Axis`×3, `FB_Spindle`, `FB_Probe`, `FB_ToolMagazine`, `FB_IO` |
| Tầng **Int** | `FB_LinearMove`, `FB_ToolChange`, `FB_Leveling`, `FB_Homing` |
| Tầng **Main** | `OB1` + `FB_Sequence` (máy trạng thái) + `FB_Comm` |

Kiến trúc 3 tầng hướng đối tượng của đồ án gốc ánh xạ rất tự nhiên sang FB/FC của TIA Portal —
đây là điểm may mắn khiến việc chuyển đổi không phải thiết kế lại từ đầu.

## B6. Giao tiếp Pi 4 ↔ PLC

`python-snap7` qua Ethernet, giao thức S7 trên nền ISO-TCP **cổng 102**, tận dụng Gigabit Ethernet
onboard của Pi 4.

> **Hai cấu hình bắt buộc trong TIA Portal.** Thiếu một trong hai thì kết nối vẫn thành công nhưng
> **đọc ra toàn số 0** — lỗi rất khó chẩn đoán:
> 1. CPU → Protection & Security → bật **"Permit access with PUT/GET communication from remote partner"**
> 2. DB trao đổi → Properties → **bỏ tick "Optimized block access"**

Cấu trúc DB và cơ chế ring buffer: xem `chuc-nang-plc.md` §5 và §7.

---

# Ghi chú về tài liệu nguồn

Bản PDF đồ án có vài chỗ không nhất quán. Ghi lại để người đọc sau không bị nhầm khi đối chiếu:

| Vấn đề | Chi tiết |
|---|---|
| Học vị GVHD | Bìa ghi **"TS."** Lê Thanh Tùng, các trang trong ghi **"ThS."** |
| Đánh số bảng | Danh mục bảng biểu ghi *"Bảng 5.2"*, nhưng bảng thực tế trong Chương 5 đánh số **"Bảng 5.1"** |
| Vùng gia công | Mục 1.6 ghi **130×180 mm**, Bảng 3.1 ghi **180×130 mm** |

---

## Tài liệu liên quan

| File | Nội dung |
|---|---|
| [`chuc-nang-plc.md`](chuc-nang-plc.md) | Đặc tả PLC: I/O, khối hàm, DB, an toàn, đấu dây |
| [`gerber-sang-nc.md`](gerber-sang-nc.md) | Sinh đường chạy dao từ file Gerber |
| [`kiem-tra-quang-hoc.md`](kiem-tra-quang-hoc.md) | Kiểm tra quang học tự động sau gia công |
| [`so-sanh-stm32-vs-s7.md`](so-sanh-stm32-vs-s7.md) | Đối chiếu bản gốc ↔ bản PLC |
