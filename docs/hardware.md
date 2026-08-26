# Máy gia công mạch PCB tự động — Hướng dẫn lắp đặt phần cứng

Tài liệu hướng dẫn quy trình lắp đặt, đấu dây và kiểm tra phần cứng trước khi cấp điện
cho bản thiết kế PLC S7-1200.

Xem `thiet-bi-va-chuc-nang.md` cho danh sách thiết bị đầy đủ.
Xem `chuc-nang-plc.md` cho bảng I/O và sơ đồ nguyên lý chi tiết.

---

## 1. Danh sách vật tư cần mua thêm

Phần cơ khí giữ nguyên từ đồ án. Phần dưới đây chỉ liệt kê thiết bị **mới** cần mua cho bản PLC.

### 1.1 Điều khiển

| Thiết bị | Model / Thông số | SL | Ghi chú |
|---|---|---|---|
| CPU PLC | **S7-1200 1214C DC/DC/DC** (6ES7214-1AG40-0XB0) | 1 | **Bắt buộc bản DC/DC/DC** — bản Rly không phát xung PTO/PWM |
| Module mở rộng | **SM 1223 DI8/DQ8 ×24VDC** (6ES7223-1BH32-0XB0) | 1 | Bù thêm 8 DO còn thiếu |
| Nguồn PLC | **Siemens PM1207 24 V / 2,5 A** (6EP1332-1SH71) | 1 | Tách biệt với động lực |
| Master | **Raspberry Pi 4 (4 GB)** | 1 | — |
| Camera | IMX219 **có ngàm M12** | 1 | **Bắt buộc bản ngàm M12** — Pi Camera Module 2/3 không lấy nét được |
| Ống kính | M12 tiêu cự **8 mm** | 1 | Đo khoảng hở trước khi mua — xem §2.3 |
| Đèn vòng | LED khuếch tán CRI cao | 1 | Không thể thay bằng đèn trực tiếp |

### 1.2 Driver và công suất

| Thiết bị | Model / Thông số | SL | Ghi chú |
|---|---|---|---|
| Driver bước | **Leadshine DM542** (ưu tiên) hoặc TB6600 | 3 | DM542: 200 kHz, opto cách ly sẵn, ghi rõ điện trở hạn dòng |
| Nguồn động lực | **Nguồn xung 24 V / 15 A (360 W)** | 1 | — |
| Hạ áp | **DC-DC 24 V → 5 V / 5 A (25 W)** | 1 | Cấp Pi 4 và LCD — đảm bảo 5 V/3 A tối thiểu |
| H-bridge spindle | **BTS7960** + board opto 24 V→5 V | 1 | — |
| Relay trung gian | Relay 24 VDC, tiếp điểm 10 A | 2 | Relay đảo chiều spindle + relay ngắt tín hiệu leveling |
| Relay an toàn | Relay an toàn (safety relay) 24 V, 2 kênh | 1 | Mạch E-stop — **không dùng relay thường** |

### 1.3 Điện trở hạn dòng — dễ bỏ sót nhất

| Linh kiện | Thông số | SL |
|---|---|---|
| **Điện trở hạn dòng** | **2 kΩ / 0,25 W** | **9** (3 trục × PUL/DIR/ENA) |

> **Thiếu điện trở là cháy opto driver ngay lần cấp điện đầu tiên.** DM542 ghi rõ: 5 V không cần ·
> 12 V thêm 1 kΩ · **24 V thêm 2 kΩ**. Xem §4.2 và `chuc-nang-plc.md` §7.2.

### 1.4 Vật tư tủ điện và cáp

| Vật tư | Thông số | Ghi chú |
|---|---|---|
| Tủ điện | Tủ kín IP54, đủ chỗ cho DIN rail 3 hàng | — |
| DIN rail | 35 mm | — |
| Terminal block | Loại 24 V / 10 A, màu khác nhau cho L+/M/PE | — |
| Aptomat chính | 2 P, 16 A | — |
| Cầu chì | 4 A (mạch điều khiển), 10 A (mạch động lực) | — |
| **Cáp xoắn đôi có lưới chắn** | 2×0,5 mm², lưới bện | PUL/DIR/ENA — **bắt buộc** |
| Cáp động cơ | 4×0,75 mm² có lưới chắn | Từ driver ra NEMA17 |
| Cáp mạng | Cat5e hoặc Cat6, đầu RJ45 | Pi 4 ↔ PLC — không cần switch |
| Cáp USB-C | 5 V/3 A có chip sạc | Cấp nguồn Pi 4 — không dùng cáp sạc điện thoại rẻ |
| Quạt thông gió tủ | 24 VDC | Điều khiển từ `Q2.4` |
| Thanh tiếp đất (PE bar) | — | Nối vỏ tủ + lưới chắn cáp |
| Tản nhiệt + quạt cho Pi 4 | Tản nhiệt gắn chip + quạt 5 V | Pi 4 throttle ở 80 °C |

---

## 2. Chuẩn bị trước khi lắp đặt

### 2.1 Kiểm tra cơ khí

Trước khi lắp bất kỳ thiết bị điện nào, xác nhận các điểm sau:

- [ ] Ba trục X/Y/Z chạy trơn bằng tay — không kẹt, không rơ quá mức
- [ ] Vitme và đai ốc đồng đã được tra dầu mỡ
- [ ] Thanh trượt và con trượt sạch, không bavia
- [ ] Ổ chứa dao (6 ổ) gắn chắc trên bàn máy, các ổ thẳng hàng
- [ ] Cụm trục chính (spindle) gắn chắc vào con trượt Z

### 2.2 Đo khoảng hở để chọn ống kính camera

Đây là bước **bắt buộc** trước khi mua ống kính M12.

Dùng thước đo khoảng cách từ **dầm ngang** (nơi gắn camera) đến **mặt bàn** khi trục Z ở vị trí
chụp ảnh (thường là Z cao hết hành trình):

| Khoảng hở đo được | Ống kính M12 chọn | FOV tại khoảng cách đó |
|---|---|---|
| ~62 mm | 4 mm | — |
| ~92 mm | 6 mm | — |
| **~123 mm** | **8 mm** | 52,9 × 39,7 mm — **mục tiêu thiết kế** |
| ~185 mm | 12 mm | — |

> Tiêu cự 8 mm là mục tiêu `[ĐX]`. Chênh lệch ±10 mm so với 123 mm vẫn cho ảnh rõ (DOF 1,98 mm).
> Nếu khoảng hở không khớp bảng trên, chọn ống kính gần nhất.

### 2.3 Chuẩn bị thẻ nhớ Raspberry Pi 4

1. Ghi Raspberry Pi OS (64-bit) vào thẻ microSD ≥ 32 GB (dùng Raspberry Pi Imager)
2. Kích hoạt SSH và cấu hình WiFi trong Imager (cho lần đầu cài đặt phần mềm)
3. Cài đặt sau khi khởi động lần đầu:
   ```bash
   sudo apt update && sudo apt upgrade -y
   pip3 install python-snap7 opencv-python-headless
   ```
4. Cài `snap7` C library: `sudo apt install libsnap7-1 libsnap7-dev`

---

## 3. Lắp ráp tủ điện

### 3.1 Bố trí DIN rail

Sắp xếp từ trên xuống dưới để đi dây gọn:

```
Hàng 1 (trên cùng):   Aptomat chính  |  Cầu chì các nhánh
Hàng 2:               PM1207 (PLC)   |  CPU 1214C  |  SM 1223
Hàng 3:               DM542-X  |  DM542-Y  |  DM542-Z
Hàng 4:               Nguồn 24V/15A  |  DC-DC 5V   |  BTS7960 + opto
Hàng 5 (thấp nhất):   Terminal block (đầu ra ngoài)
```

Relay an toàn E-stop đặt gần aptomat để đường cắt điện ngắn.

### 3.2 Thứ tự lắp đặt

Lắp theo thứ tự này để tránh phải tháo ra làm lại:

1. **Gắn DIN rail** vào tủ
2. **Gắn thiết bị** lên DIN rail (chưa đấu dây)
3. **Gắn terminal block** và thanh PE
4. **Đi dây động lực** (nguồn AC → aptomat → nguồn xung, PM1207)
5. **Đi dây điều khiển** (PLC I/O, relay, cảm biến)
6. **Đi cáp lưới chắn** (PUL/DIR, cáp động cơ)
7. **Kiểm tra trước khi cấp điện** (xem §5)

---

## 4. Đấu nối từng cụm

### 4.1 Nguồn — hai nhánh tách biệt

```
220 VAC vào aptomat
    ├─→  Nguồn xung 24 V / 15 A  →  thanh L+ và M (động lực)
    │       ↓
    │    Tiếp điểm chính relay an toàn E-stop nằm giữa nguồn và thanh phân phối động lực
    │
    └─→  PM1207 24 V / 2,5 A  →  thanh L+ và M riêng (PLC/điều khiển)
             → CPU 1214C (L1, M1)
             → SM 1223 (cùng ray với CPU, tự lấy qua backplane bus)
```

> **Không được nối chung hai thanh M (0 V).** Mục đích của hai nguồn là cách ly nhiễu.
> Nếu nối chung thì mạch phân ly không còn tác dụng.

### 4.2 PLC → Driver bước (3 trục)

Đây là điểm **quan trọng nhất về đấu nối** — sai kiểu đấu hoặc thiếu điện trở là hỏng ngay.

Áp dụng cho cả ba trục (thay Q0.0/Q0.1 bằng Q0.2/Q0.3 và Q0.4/Q0.5 cho trục Y, Z):

```
Trục X — DM542-X
─────────────────────────────────────────────────────────────
S7-1200 Q0.0  ──[ 2 kΩ ]──────────────→  PUL+  ┐
S7-1200 Q0.1  ──[ 2 kΩ ]──────────────→  DIR+  │  DM542-X
S7-1200 Q0.7  ──[ 2 kΩ ]──────────────→  ENA+  │
S7-1200 M     ─────────────────────────→  PUL-  │
              (0 V)                    →  DIR-  │
                                       →  ENA-  ┘

Nguồn 24 V / 15 A  (+)  ───────────────→  VCC
Nguồn 24 V / 15 A  (−)  ───────────────→  GND
─────────────────────────────────────────────────────────────
DM542-X  →  NEMA17-X  (4 dây A+, A−, B+, B−, cáp có lưới chắn)
```

**Quy tắc kiểu đấu:**

| Điểm | Chi tiết |
|---|---|
| Chung âm (common cathode) | Ngõ ra S7-1200 DC/DC/DC là sourcing (PNP) — cổng Q đóng +24 V. Nối `PUL−/DIR−/ENA−` về chân M của PLC |
| 2 kΩ trên mỗi đường tín hiệu | Opto ngõ vào driver thiết kế cho 5 V. Tổng 9 điện trở cho 3 trục |
| DIP switch dòng | Đặt **1,4–1,6 A đỉnh** (NEMA17 định mức 1,6 A) |
| DIP switch vi bước | Đặt **1/16** → 3200 xung/vòng → độ phân giải 2,5 µm/xung |

### 4.3 PLC → BTS7960 (spindle)

BTS7960 dùng logic 3,3–5 V, PLC ra 24 V → **bắt buộc board opto 24 V→5 V ở giữa**.

| Từ PLC | Qua board opto | Vào BTS7960 |
|---|---|---|
| `Q0.6` (PWM4) | opto 24 V→5 V | `RPWM` — tốc độ spindle |
| `Q1.0` (relay đảo chiều) | relay chuyển mạch | `LPWM` — quay ngược cho ATC |
| `Q2.0` (SM 1223) | opto 24 V→5 V | `R_EN` + `L_EN` — bật/tắt cầu H |

BTS7960 cấp nguồn trực tiếp từ thanh 24 V động lực. Động cơ RS775 đấu vào đầu ra M+ / M−.

### 4.4 Cảm biến và nút điều khiển

| Thiết bị | Chân PLC | Loại tiếp điểm | Ghi chú |
|---|---|---|---|
| Công tắc gốc X | `I0.0` | NO | Cam cho `MC_Home` |
| Công tắc gốc Y | `I0.1` | NO | — |
| Công tắc gốc Z | `I0.2` | NO | — |
| **Probe leveling** | **`I0.3`** | NO | **Khai báo Hardware Interrupt cạnh lên trong TIA Portal** |
| Hard limit X | `I0.4` | NC | Mở mạch khi đụng cuối hành trình |
| Hard limit Y | `I0.5` | NC | — |
| Hard limit Z | `I0.6` | NC | — |
| Tiếp điểm phụ E-stop | `I0.7` | NC | Chỉ để PLC biết trạng thái — **không phải đường cắt** |
| Start | `I1.0` | NO | — |
| Stop | `I1.1` | NO | — |
| Reset | `I1.2` | NO | — |
| Alarm driver X | `I1.3` | NC | Mất bước hoặc quá nhiệt |
| Alarm driver Y | `I1.4` | NC | — |
| Alarm driver Z | `I1.5` | NC | — |

Tất cả cảm biến và nút chịu điện áp **24 VDC** từ thanh L+/M của PM1207.

### 4.5 Mạch E-stop

```
Nút E-stop (NC) ─── Relay an toàn ─── Tiếp điểm chính ─── Thanh phân phối 24 V động lực
                                             │
                                             ↓ (cắt cứng)
                              Driver bước × 3, BTS7960 → Spindle

Tiếp điểm phụ ────────────────────────────→ I0.7 (PLC chỉ đọc)
```

> **Không đi E-stop qua PLC.** Relay an toàn cắt thẳng nguồn động lực — không phụ thuộc chương
> trình PLC. PLC chỉ đọc tiếp điểm phụ `I0.7` để hiển thị và khóa lệnh tiếp theo.

### 4.6 Kết nối Raspberry Pi 4

| Kết nối | Phương tiện | Ghi chú |
|---|---|---|
| Nguồn | USB-C, 5 V/3 A | **Cấp qua USB-C** — không cấp thẳng vào header 40 chân trừ khi có cầu chì và kiểm tra vọt áp |
| Camera | CSI (flat cable 15 chân) | Cáp bọc kim nếu khoảng cách > 200 mm |
| LCD 3.5" | SPI + GPIO | Xem hướng dẫn từng model LCD |
| PLC | Ethernet RJ45 Cat5e | Đấu thẳng dây — cổng PROFINET có auto-crossover |
| USB Gerber | USB 3.0 | — |

---

## 5. Kiểm tra trước khi cấp điện

**Hoàn thành toàn bộ checklist này trước khi đóng aptomat.**

### 5.1 Kiểm tra nguội (không điện)

**Nguồn:**
- [ ] Đo điện trở giữa L+ và M của nguồn 24 V / 15 A — **không được gần 0 Ω** (ngắn mạch)
- [ ] Đo điện trở giữa L+ và M của PM1207 — tương tự
- [ ] PE nối đất tủ, nối vỏ thiết bị kim loại, nối lưới chắn cáp **một đầu** phía tủ
- [ ] Aptomat đang **MỞ** (OFF)

**Đấu nối PLC → Driver:**
- [ ] Xác nhận **9 điện trở 2 kΩ** đã lắp (PUL/DIR/ENA × 3 trục)
- [ ] Xác nhận chung âm: `PUL−`, `DIR−`, `ENA−` nối về chân M của PLC
- [ ] DIP switch driver: dòng 1,4–1,6 A, vi bước 1/16

**Cảm biến:**
- [ ] Probe leveling đấu vào `I0.3` — kiểm tra khai báo Hardware Interrupt trong TIA Portal
- [ ] Hard limit đấu công tắc NC (mạch kín khi bình thường)
- [ ] Relay an toàn E-stop đấu đúng — tiếp điểm chính nằm trong đường cấp 24 V động lực

**BTS7960:**
- [ ] Board opto 24 V→5 V đã lắp giữa PLC và BTS7960
- [ ] Động cơ RS775 đấu vào M+/M− đúng chiều quay mong muốn

### 5.2 Kiểm tra khi cấp điện lần đầu — từng bước

Cấp điện theo thứ tự, kiểm tra từng bước trước khi tiếp tục:

**Bước 1 — Đóng aptomat, chưa kết nối bất kỳ thiết bị tiêu thụ nào:**
- Đo `V(L+) − V(M)` nguồn xung: **24 V ± 5%**
- Đo `V(L+) − V(M)` PM1207: **24 V ± 2%**

**Bước 2 — Cấp điện CPU S7-1200:**
- Đèn `RUN` (xanh) sáng — CPU đang chạy
- Đèn `ERROR` (đỏ) không sáng
- Nếu `ERROR` sáng: kết nối TIA Portal → xem Online diagnostics

**Bước 3 — Cấp điện Raspberry Pi 4:**
- Pi 4 khởi động (đèn LED xanh nhấp nháy)
- Đo điện áp 5 V tại Pi: **4,9–5,1 V** — nếu thấp hơn 4,75 V, kiểm tra lại bộ chuyển DC-DC hoặc cáp USB-C

**Bước 4 — Thử từng driver bước (chưa kết nối động cơ):**
- Cấp điện một driver DM542
- Đèn `POWER` sáng, đèn `ALARM` tắt
- Tắt trước khi sang driver tiếp theo

**Bước 5 — Kết nối động cơ và thử quay từ TIA Portal:**
- Dùng **Force I/O** để phát xung kiểm tra từng trục — **vận tốc thấp**
- Quan sát chiều quay: bàn máy di chuyển đúng trục và đúng chiều

---

## 6. Cấu hình TIA Portal — các điểm bắt buộc

Sau khi phần cứng đã cấp điện và CPU sống, thực hiện các cấu hình này trong TIA Portal trước
khi nạp chương trình lần đầu.

### 6.1 Cấu hình trục (Technology Object)

Với mỗi trục X/Y/Z, tạo `TO_PositioningAxis` và đặt:

| Thông số | Giá trị |
|---|---|
| Drive type | PTO |
| PTO output | Q0.0/Q0.1 (X), Q0.2/Q0.3 (Y), Q0.4/Q0.5 (Z) |
| Pulse per revolution | **3200** (vi bước 1/16) |
| Lead screw pitch | **8 mm** |
| Max velocity | **25 mm/s** (dư biên so với 20 mm/s gia công) |
| Max acceleration | **50 mm/s²** `[ĐX]` |
| Software limit (positive) | X: 220 mm · Y: 235 mm · Z: 50 mm |
| Software limit (negative) | 0 mm cho cả ba |
| Homing mode | **Active homing (Mode 3)** — dùng công tắc gốc làm cam |

### 6.2 Hardware Interrupt cho probe leveling

1. Device configuration → CPU → Digital inputs → `I0.3`
2. Enable **Hardware interrupt** → Rising edge (cạnh lên)
3. Gán tới OB Hardware Interrupt đã khai báo trong `FB_Leveling`

> **Không bỏ bước này.** Nếu probe chỉ được quét trong `OB1`, sai số Z sẽ bằng một chu kỳ quét
> (thường 1–10 ms). Ở tốc độ dò chậm 1 mm/s, chu kỳ 5 ms → sai số 5 µm — đủ để phay lẹm vào đồng.

### 6.3 Cho phép PUT/GET và tắt Optimized block access

Hai cấu hình bắt buộc để `python-snap7` đọc/ghi được DB:

1. CPU → Properties → **Protection & Security** → tích **"Permit access with PUT/GET communication from remote partner"**
2. Với mỗi DB trao đổi (`DB_Command`, `DB_Status`, `DB_Buffer`) → Properties → **bỏ tick "Optimized block access"**

> Thiếu một trong hai thì kết nối snap7 vẫn thành công nhưng **đọc ra toàn số 0** — lỗi rất khó
> chẩn đoán vì không có thông báo lỗi rõ ràng.

### 6.4 Địa chỉ IP

| Thiết bị | Địa chỉ IP gợi ý |
|---|---|
| S7-1200 (PROFINET) | `192.168.0.1` |
| Raspberry Pi 4 | `192.168.0.100` (static) |

Đặt IP tĩnh trên Pi 4 qua `/etc/dhcpcd.conf`:
```
interface eth0
static ip_address=192.168.0.100/24
```

---

## 7. Kiểm tra chống nhiễu (EMC)

Đây là nguyên nhân hàng đầu gây lỗi khó giải thích sau khi lắp đặt xong.

### 7.1 Kiểm tra đi cáp

- [ ] Cáp PUL/DIR/ENA là **cáp xoắn đôi có lưới chắn**, lưới nối PE **một đầu** phía tủ điện
- [ ] Cáp động cơ (driver → NEMA17) có lưới chắn, nối PE một đầu phía tủ
- [ ] Cáp tín hiệu (cảm biến, nút) cách cáp động lực (pha động cơ) **ít nhất 10 cm**
- [ ] Nơi hai loại cáp giao nhau: **vuông góc 90°**, không chạy song song
- [ ] Cáp Ethernet Pi↔PLC không đi song song với cáp động cơ

### 7.2 Triệu chứng nhiễu và cách kiểm tra

| Triệu chứng | Nguyên nhân nhiễu có thể | Cách kiểm tra |
|---|---|---|
| Trục chạy sai vị trí hoặc mất bước nhưng không alarm | Nhiễu trên PUL — driver đếm nhầm xung | Dùng oscilloscope đo dạng sóng PUL+ tại đầu vào driver khi spindle chạy |
| Probe leveling báo chạm nhưng dao chưa chạm | Nhiễu trên `I0.3` | Thêm RC filter (10 kΩ + 100 nF) tại đầu vào I0.3 |
| CPU reset hoặc đọc sai ngõ vào khi spindle chạy | Chung nguồn hoặc nhiễu trên đường nguồn PLC | Kiểm tra PM1207 đã tách nguồn hoàn toàn khỏi nguồn động lực |
| snap7 kết nối được nhưng đọc nhầm giá trị | Nhiễu Ethernet từ driver | Đảm bảo cáp Ethernet tránh xa cáp driver; dùng Cat5e STP nếu còn vấn đề |

---

## 8. Thông số thực nghiệm cần đo sau khi lắp đặt

Các thông số sau **không thể tính từ lý thuyết** — phải đo trên máy thực.

| Thông số | Lý do phải đo | Cách đo |
|---|---|---|
| **PWM siết dao ATC ở 24 V** | Đồ án đo ở 12 V — **không dùng được** | Thực nghiệm 5 lần, ghi PWM duty và thời gian đặt siết/nhả đáng tin cậy |
| **PWM nhả dao ATC ở 24 V** | Tương tự | Tương tự |
| **Thời gian ổn định sau dừng** | Ảnh hưởng chất lượng AOI | Chụp ảnh ở các mức delay 0,2 / 0,5 / 1,0 s sau `MC_MoveAbsolute Done`, chọn giá trị nhỏ nhất cho ảnh không nhòe |
| **Thời gian suy luận YOLO** | Quyết định chu kỳ kiểm tra | Chạy trên Pi 4 thực tế, đo `time.perf_counter()` trước và sau `model.predict()` |
| **Leveling map 66 điểm** | Bề mặt bàn máy mỗi máy khác nhau | Chạy quy trình leveling đầy đủ, lưu ma trận Z vào file |

---

## 9. Tài liệu liên quan

| File | Nội dung |
|---|---|
| [`thiet-bi-va-chuc-nang.md`](thiet-bi-va-chuc-nang.md) | BOM đầy đủ và danh sách chức năng |
| [`chuc-nang-plc.md`](chuc-nang-plc.md) | Bảng I/O đầy đủ, khối hàm, DB, sơ đồ nguyên lý |
| [`gerber-sang-nc.md`](gerber-sang-nc.md) | Sinh đường chạy dao từ Gerber |
| [`kiem-tra-quang-hoc.md`](kiem-tra-quang-hoc.md) | Kiểm tra quang học tự động (AOI) |
| [`luu-do-giai-thuat.md`](luu-do-giai-thuat.md) | Lưu đồ giải thuật toàn hệ thống |
| [`dinh-vi-va-toi-uu-phoi.md`](dinh-vi-va-toi-uu-phoi.md) | Định vị phôi và tối ưu tận dụng phôi thừa |
| [`so-sanh-stm32-vs-s7.md`](so-sanh-stm32-vs-s7.md) | Đối chiếu bản STM32 gốc ↔ bản PLC |
