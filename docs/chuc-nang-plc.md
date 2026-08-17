# Đặc tả chức năng PLC S7-1200

Tài liệu để lập trình TIA Portal và đấu tủ điện cho máy gia công mạch PCB tự động.
Xem `thiet-bi-va-chuc-nang.md` cho danh sách thiết bị tổng thể.

**Cấu hình phần cứng:** CPU **S7-1200 1214C DC/DC/DC** + **SM 1223 DI8/DQ8 ×24VDC**

> **Bắt buộc bản DC/DC/DC.** Bản ngõ ra relay (DC/DC/Rly) không phát xung PTO/PWM được.

---

## 1. Ranh giới trách nhiệm Pi 4 ↔ PLC

Đây là quyết định thiết kế quan trọng nhất của cả hệ thống. Đặt sai ranh giới thì chương trình PLC
phình to, mất tính tất định, và phải chống chọi với điểm yếu cố hữu của PLC là xử lý chuỗi và dữ liệu lớn.

| Raspberry Pi 4 — Master | S7-1200 — Slave |
|---|---|
| Đọc USB, chuyển Gerber + Excellon → `.nc` | **Không** đụng tới file |
| Phân tích cú pháp G-code, tuyến tính hóa cung tròn | **Không** phân tích cú pháp |
| Tính leveling map (66 điểm → phương trình mặt phẳng) và **bù z sẵn vào từng lệnh** | **Không** tính nội suy mặt phẳng |
| Chia nhỏ đoạn dài để z bám bề mặt | **Không** chia đoạn |
| Sắp thứ tự đường chạy, tối ưu quãng di chuyển | **Không** tối ưu quỹ đạo |
| Giao diện, lưu trữ, mô phỏng Preview | **Không** có UI |
| **Toàn bộ xử lý ảnh AOI**: ghép ảnh, phân đoạn, sinh ứng viên, suy luận YOLO | **Chỉ** di chuyển tới vị trí chụp và báo đã tới nơi |
| | **Thực thi** chuyển động, I/O, ATC, an toàn, dò probe |

**Nguyên tắc:** PLC chỉ nhận lệnh **đã sẵn sàng chạy** — tọa độ tuyệt đối, z đã bù, tốc độ đã tính.
Nhờ vậy chương trình PLC gọn, tất định, và **không phụ thuộc vào định dạng file** — sau này đổi
định dạng file chỉ phải sửa phía Pi.

---

## 2. Bảng phân bổ I/O

### 2.1 Ngõ ra tốc độ cao — bắt buộc onboard

> **Ràng buộc cứng:** kênh PTO/PWM **chỉ hoạt động trên ngõ ra onboard `Q0.0`–`Q0.7`**.
> Đặt trên SM 1223 là **không phát xung được**. Đây là gán mặc định của TIA Portal.

| Kênh | Địa chỉ | Chức năng |
|---|---|---|
| **PTO1** | `Q0.0` / `Q0.1` | PUL / DIR trục **X** |
| **PTO2** | `Q0.2` / `Q0.3` | PUL / DIR trục **Y** |
| **PTO3** | `Q0.4` / `Q0.5` | PUL / DIR trục **Z** |
| **PWM4** | `Q0.6` | Tốc độ spindle → opto 24→5 V → BTS7960 |

Bốn bộ phát xung dùng hết — đây chính là lý do chọn S7-1200 thay vì S7-200 SMART (chỉ có 3 kênh,
không đủ cho cả 3 trục lẫn spindle).

### 2.2 Ngõ ra thường

| Địa chỉ | Nguồn | Chức năng |
|---|---|---|
| `Q0.7` | Onboard | ENA chung cho 3 driver bước |
| `Q1.0` | Onboard | Relay đảo chiều spindle (nhả dao ATC) |
| `Q1.1` | Onboard | Relay ngắt tín hiệu leveling khi spindle quay |
| `Q2.0` | SM 1223 | Enable BTS7960 (`R_EN` + `L_EN`) |
| `Q2.1` | SM 1223 | Đèn báo chạy (xanh) |
| `Q2.2` | SM 1223 | Đèn báo lỗi (đỏ) |
| `Q2.3` | SM 1223 | Còi báo |
| `Q2.4` | SM 1223 | Quạt thông gió tủ điện |
| `Q2.5`–`Q2.7` | SM 1223 | Dự phòng (module phủ keo UV) |

### 2.3 Ngõ vào

| Địa chỉ | Chức năng | Ghi chú |
|---|---|---|
| `I0.0` / `I0.1` / `I0.2` | Công tắc gốc X / Y / Z | Làm cam cho `MC_Home` |
| **`I0.3`** | **Probe leveling** | **Bật ngắt phần cứng cạnh lên** |
| `I0.4` / `I0.5` / `I0.6` | Hard limit X / Y / Z | |
| `I0.7` | Tiếp điểm phụ E-stop | Chỉ để PLC biết trạng thái |
| `I1.0` / `I1.1` / `I1.2` | Start / Stop / Reset | |
| `I1.3` / `I1.4` / `I1.5` | Alarm driver X / Y / Z | Mất bước, quá nhiệt |
| `I3.0`–`I3.7` | Dự phòng | Cảm biến có dao, cửa an toàn… |

> **Probe leveling bắt buộc dùng ngắt phần cứng.** Nếu chỉ quét trong `OB1`, sai số Z sẽ bằng cả
> một chu kỳ quét. Với dao phay PCB ăn sâu vài chục µm, đó là hỏng mạch. Ngắt phần cứng cho độ trễ
> cỡ µs — ở tốc độ dò chậm thì sai số không đáng kể.

### 2.4 Tổng kết sử dụng

| | Đã dùng | Tổng có | Dư |
|---|---|---|---|
| **DO** | **15** (10 onboard + 5 SM) | 18 (10 onboard + 8 SM) | **3** — `Q2.5`–`Q2.7` |
| **DI** | **14** (14 onboard + 0 SM) | 22 (14 onboard + 8 SM) | **8** — `I3.0`–`I3.7` |

> **Ngõ ra onboard đã dùng hết 10/10.** Mọi chức năng bổ sung bắt buộc phải nằm trên SM 1223, mà
> ngõ ra SM **không phát xung được** — nên nếu sau này cần thêm trục hoặc thêm kênh PWM thì phải
> đổi CPU chứ không thể mở rộng bằng module.
>
> Module phủ keo UV còn 3 DO và 8 DI: đủ cho một van, một bơm và một đèn báo. Nếu cần nhiều hơn
> thì thêm SM 1223 thứ hai.

> **Địa chỉ SM là ví dụ.** TIA Portal tự gán địa chỉ cho module mở rộng theo slot và cho phép sửa
> trong Device configuration. Các địa chỉ `Q2.x` / `I3.x` ở trên là minh họa — **phải đối chiếu với
> cấu hình thực tế của dự án**, không chép cứng.

---

## 3. Danh sách khối hàm

Ánh xạ kiến trúc 3 tầng **Firmware_ADL** của đồ án gốc sang FB/FC của TIA Portal. Kiến trúc gốc
vốn đã hướng đối tượng nên chuyển đổi rất tự nhiên.

### 3.1 Tầng Unit — thay `Unit_*`

Quản lý từng thiết bị đơn lẻ, tương ứng khái niệm "ngoại vi" của đồ án gốc.

| FB | Chức năng |
|---|---|
| `FB_Axis` ×3 | Bọc `TO_PositioningAxis`: cấp/ngắt trục, về gốc, di chuyển, dừng, đọc vị trí, gom lỗi |
| `FB_Spindle` | Đặt tốc độ qua PWM, đảo chiều qua relay, enable BTS7960, định thời siết/nhả dao |
| `FB_Probe` | Quản lý tín hiệu leveling + đóng/mở relay chống nhiễu |
| `FB_ToolMagazine` | Tọa độ 6 ổ dao, theo dõi dao đang gắn trên spindle |
| `FB_IO` | Lọc rung nút nhấn, điều khiển đèn và còi |

### 3.2 Tầng Int — thay `Int_*`

Ghép các Unit theo một luật điều khiển. Đúng định nghĩa "Integration" của đồ án gốc.

| FB | Chức năng |
|---|---|
| `FB_LinearMove` | **Nội suy đường thẳng 2 trục** — xem §4.2 |
| `FB_ToolChange` | Chuỗi ATC đầy đủ — xem §4.3 |
| `FB_Leveling` | Dò 1 điểm: hạ Z chậm bằng `MC_MoveVelocity` → ngắt phần cứng → `MC_Halt` → chốt Z |
| `FB_Homing` | Về gốc theo thứ tự **Z → X → Y** |

> **Thứ tự về gốc Z → X → Y không được đổi.** Z phải lên hết trước, nếu không dao còn cắm trong
> phôi mà bàn máy chạy X/Y là gãy dao và xước bo.

### 3.3 Tầng Main — thay `Main_*`

| Khối | Chức năng |
|---|---|
| `OB1` | Chu kỳ chính, gọi `FB_Sequence` và `FB_Comm` |
| `FB_Sequence` | Máy trạng thái: `IDLE → HOMING → LEVELING → READY → RUNNING → INSPECTING → PAUSED → ERROR` |
| `FB_Comm` | Giải mã `DB_Command`, cập nhật `DB_Status`, quản lý ring buffer và bắt tay |

### 3.4 OB đặc biệt

| OB | Chức năng |
|---|---|
| **Hardware Interrupt** (`I0.3`) | Bắt chạm probe leveling — quyết định độ chính xác Z |
| `OB100` Startup | Khởi tạo, xóa trạng thái, **khóa chạy đến khi về gốc xong** |
| `OB82` Diagnostic error | Lỗi chẩn đoán module |
| `OB121` / `OB122` | Lỗi lập trình / lỗi truy cập I/O |

---

## 4. Chuyển động

### 4.1 Lệnh Motion Control sử dụng

| Lệnh | Dùng cho |
|---|---|
| `MC_Power` | Cấp/ngắt trục — ánh xạ nút "Disable/Enable stepper" |
| `MC_Home` | Về gốc, **Mode 3 (active homing)** dùng công tắc làm cam |
| `MC_MoveAbsolute` | Di chuyển tới tọa độ tuyệt đối — lệnh chính khi gia công |
| `MC_MoveRelative` | Di chuyển tương đối |
| `MC_MoveVelocity` | Chạy theo vận tốc — dùng khi dò probe leveling |
| `MC_MoveJog` | Chạy tay từ màn hình |
| `MC_Halt` | Dừng có điều khiển |
| `MC_Reset` | Xóa lỗi trục |
| `MC_WriteParam` | **Đổi vận tốc từng trục để nội suy** |

### 4.2 Nội suy đường thẳng 2 trục — `FB_LinearMove`

S7-1200 **không có nội suy quỹ đạo**; các trục PTO chạy độc lập. Cách tạo đường thẳng:

```
L   = √(dx² + dy²)
v_x = v · |dx| / L
v_y = v · |dy| / L
```

Ghi hai vận tốc bằng `MC_WriteParam`, rồi phát **đồng thời** 2 lệnh `MC_MoveAbsolute`.
Hai trục xuất phát cùng lúc và về đích cùng lúc → quỹ đạo là đường thẳng.
Chờ cả hai cờ `Done` mới nhận lệnh kế.

**Hạn chế cần biết:** cách này chỉ dựng được **đường thẳng**, không có cung tròn. Với gia công PCB
thì đủ, vì cung tròn `G02`/`G03` đã được tuyến tính hóa trên Pi (xem `gerber-sang-nc.md` §6).

### 4.3 Chuỗi thay dao tự động — `FB_ToolChange`

```
1. Nâng Z về vị trí an toàn
2. Di chuyển tới ổ dao ĐANG giữ dao hiện tại
3. Hạ Z vào ổ
4. Spindle đảo chiều → nhả dao          ← thông số PWM phải đo lại ở 24 V
5. Nâng Z thoát khỏi dao
6. Di chuyển tới ổ dao MỚI
7. Hạ Z vào dao
8. Spindle quay thuận → siết dao        ← thông số PWM phải đo lại ở 24 V
9. Nâng Z, cập nhật FB_ToolMagazine
```

> **Thông số PWM siết/nhả của đồ án gốc (200/200 ms và 250/250 ms) đo ở 12 V, không dùng được ở
> 24 V.** Phải lặp lại quy trình thực nghiệm 5 lần/bộ thông số như đồ án đã làm.

### 4.4 Chuỗi chụp ảnh kiểm tra — trạng thái `INSPECTING`

Sau khi gia công xong, PLC chuyển sang trạng thái `INSPECTING`. **PLC không xử lý ảnh** — nó chỉ
đóng vai trò bàn định vị cho camera:

```
1. Nâng Z về chiều cao lấy nét đã hiệu chuẩn
2. Tắt spindle (M05) — bắt buộc, tránh rung khi chụp
3. Lặp cho k = 1..20:
      a. Nhận lệnh cmd_id = 9 (goto_capture) với tọa độ tile thứ k
      b. MC_MoveAbsolute tới (x_k, y_k)
      c. Chờ Done, thêm thời gian ổn định (~0,5 s) để tắt dao động cơ khí
      d. Đặt cờ done trong DB_Status → Pi chụp ảnh
      e. Chờ Pi xác nhận đã chụp xong qua int_sys
4. Về vị trí chờ, chuyển trạng thái READY
```

> **Thời gian ổn định sau khi dừng là bắt buộc.** Bàn máy còn dao động cơ khí vài trăm mili-giây sau
> khi trục báo `Done`. Chụp ngay lúc đó cho ảnh nhòe, mà nhòe ở mức vài chục µm là đủ phá hỏng khả
> năng phát hiện khuyết tật 0,1 mm.

> **Bắt buộc tắt spindle trước khi chụp.** Ngoài rung động, spindle quay còn gây nhiễu điện lên
> đường tín hiệu — cùng lý do đã phải có relay ngắt tín hiệu leveling ở `Q1.1`.

Độ chính xác định vị của máy (2,5 µm) nhỏ hơn GSD của camera (13,3 µm) một bậc, nên tọa độ máy tại mỗi
lần chụp **chính là thông tin đăng ký ảnh** — xem `kiem-tra-quang-hoc.md` §5.2.

---

## 5. Cấu trúc Data Block

> **Cả hai DB phải bỏ tick "Optimized block access".** Nếu không, snap7 không truy cập được —
> mà biểu hiện là **kết nối thành công nhưng đọc ra toàn số 0**, rất khó chẩn đoán.

### 5.1 `DB_Command` — Pi ghi, PLC đọc

| Biến | Kiểu | Ý nghĩa |
|---|---|---|
| `cmd_id` | `Int` | 0 none · 1 line · 2 drill · 3 cut · 4 toolchange · 5 home · 6 level · 7 jog · 8 stop · **9 goto_capture** |
| `x`, `y`, `z` | `DInt` | Tọa độ đích, đơn vị **µm**. `z` **đã được Pi bù leveling map** |
| `feed` | `Int` | Tốc độ, mm/s × 10 |
| `tool` | `Int` | Số ổ dao 1–6 |
| `int_sys` | `Int` | Bộ đếm lệnh từ Pi |
| `flags` | `Word` | Cờ phụ |

### 5.2 `DB_Status` — PLC ghi, Pi đọc

| Biến | Kiểu | Ý nghĩa |
|---|---|---|
| `state` | `Int` | Trạng thái máy (theo `FB_Sequence`) |
| `act_x`, `act_y`, `act_z` | `DInt` | Vị trí thực, µm |
| `int_cnc` | `Int` | Bộ đếm phản hồi |
| `error_code` | `Int` | Mã lỗi |
| `tool_current` | `Int` | Dao đang gắn trên spindle |
| `z_probe` | `DInt` | Kết quả dò leveling, µm |
| `busy` / `done` / `error` | `Bool` | Cờ trạng thái |

### 5.3 `DB_Buffer` — ring buffer 32 đoạn

Xem §7.1 để hiểu vì sao bắt buộc phải có. Mỗi ô chứa cùng cấu trúc như `DB_Command`.

---

## 6. Chức năng an toàn

| Chức năng | Hiện thực |
|---|---|
| **E-stop** | **Đấu cứng** qua relay an toàn, cắt thẳng nguồn 24 V động lực. **Không đi qua PLC.** PLC chỉ đọc tiếp điểm phụ `I0.7` để hiển thị và khóa chương trình |
| **Hard limit** | Vào DI của PLC **và** khuyến nghị cắt cứng chân ENA driver |
| **Watchdog truyền thông** | Mất phản hồi từ Pi quá thời gian `T` → PLC tự `MC_Halt` toàn trục, chuyển `ERROR` |
| **Khóa khi chưa về gốc** | Chặn mọi lệnh gia công cho đến khi `HOMING` hoàn tất |
| **Giám sát driver** | Đọc chân alarm 3 driver → dừng ngay khi mất bước hoặc quá nhiệt |
| **Giới hạn mềm** | Đặt trong `TO_PositioningAxis` theo hành trình 220 × 235 × 50 mm |

> **E-stop không đi qua PLC là nguyên tắc an toàn, không phải lựa chọn thiết kế.** PLC có thể treo,
> lỗi chương trình, hoặc mất nguồn logic — mạch dừng khẩn phải hoạt động độc lập với nó.

---

## 7. Chuỗi tín hiệu và đấu nối

Chỉ có **duy nhất chặng Pi → PLC** là giao tiếp theo nghĩa giao thức. Toàn bộ phần còn lại là tín
hiệu đấu cứng — đó là lý do PLC tất định và đáng tin, nhưng cũng là lý do **lỗi vận hành hầu hết
nằm ở đấu dây chứ không ở code**.

### 7.1 Pi 4 → PLC — Ethernet, giao thức S7

**Vật lý:** cáp Cat5e/Cat6, RJ45. Gigabit Ethernet của Pi 4 ↔ cổng PROFINET của S7-1200 (10/100 Mbps).
Cổng PROFINET có **auto-crossover** → **đấu thẳng dây, không cần switch**.

**Giao thức:** S7 Communication trên nền ISO-TCP (RFC 1006), **cổng TCP 102**, thư viện `python-snap7`.

```python
client.connect('192.168.0.1', 0, 1)     # rack 0, slot 1 cho S7-1200
client.db_write(1, 0, data)             # ghi DB_Command
status = client.db_read(2, 0, 24)       # đọc DB_Status
```

**Hai cấu hình bắt buộc trong TIA Portal:**

1. CPU → Protection & Security → bật **"Permit access with PUT/GET communication from remote partner"**
2. DB trao đổi → Properties → **bỏ tick "Optimized block access"**

> **Độ trễ buộc phải có ring buffer.** Mỗi lượt đọc/ghi mất **5–20 ms** → trần chỉ **50–200 lệnh/s**.
> Với đường mạch nhiều đoạn ngắn sẽ nghẽn. Giải pháp: Pi ghi **một khối 32 đoạn** vào `DB_Buffer`,
> PLC tiêu thụ dần, Pi nạp tiếp khi trống. Bắt tay `int_sys`/`int_cnc` đếm **theo ô buffer** thay vì
> theo từng lệnh.
>
> Đây là lý do thứ hai — độc lập với chuyện PTO không nội suy — khiến chương trình gia công
> **bắt buộc** phải ở dạng đoạn thẳng vector chứ không thể ở dạng từng điểm ảnh.

**Phương án thay thế** nếu không muốn mở PUT/GET (nó làm yếu bảo mật CPU):

| Phương án | Ưu | Nhược |
|---|---|---|
| **Modbus TCP** | Chuẩn mở, không cần PUT/GET. S7-1200 có sẵn lệnh `MB_SERVER`, Pi dùng `pymodbus` | Phải tự map thanh ghi |
| **OPC UA** | Hiện đại, bảo mật tốt nhất | **Cần mua license** cho S7-1200 |

### 7.2 PLC → Driver bước — xung/hướng

Không phải truyền thông mà là **tín hiệu xung/hướng đấu cứng**, mỗi trục 3 cặp dây:

| Tín hiệu | Nguồn | Chức năng |
|---|---|---|
| `PUL+` / `PUL−` | PTO (`Q0.0` / `Q0.2` / `Q0.4`) | Mỗi xung = 1 vi bước |
| `DIR+` / `DIR−` | `Q0.1` / `Q0.3` / `Q0.5` | Mức cao/thấp = chiều quay |
| `ENA+` / `ENA−` | `Q0.7` (chung 3 trục) | Cấp/ngắt dòng giữ động cơ |

**Kiểu đấu: chung âm (common cathode).**

Ngõ ra S7-1200 DC/DC/DC là **sourcing (PNP)** — đóng +24 V xuống tải. Do đó:

```
PUL− , DIR− , ENA−  →  chân M (0 V) của PLC
PUL+ , DIR+ , ENA+  →  chân Q của PLC  (qua điện trở hạn dòng)
```

> **Đấu ngược kiểu là driver không nhận xung.** Đây là lỗi rất hay gặp khi chuyển từ vi điều khiển
> sang PLC, vì DRV8825 trước đây nhận logic 3,3 V kiểu khác hẳn.

> **Điện trở hạn dòng — chi tiết dễ bỏ sót nhất, bỏ qua là cháy opto driver.**
> Ngõ vào driver là optocoupler thiết kế cho 5 V. Với 24 V phải mắc **nối tiếp ~2 kΩ / 0,25 W**
> trên **mỗi** đường tín hiệu (9 điện trở cho 3 trục).
> Leadshine DM542 ghi rõ: 5 V không cần · 12 V thêm 1 kΩ · **24 V thêm 2 kΩ**.
> Một số TB6600 nhận thẳng 24 V — **phải tra datasheet đúng model bạn mua, không suy đoán**.

**Kiểm tra biên độ tần số:**

| Chỉ tiêu | Hệ cần | DM542 | TB6600 |
|---|---|---|---|
| Tần số xung | **8 kHz** | 200 kHz | ~20 kHz |
| Độ rộng xung | **62,5 µs** | ≥ 2,5 µs | — |

Cả hai đều dư biên. Nếu sau này chuyển vi bước 1/32 (16 kHz) thì DM542 còn nhiều biên hơn hẳn.

### 7.3 Chống nhiễu (EMC)

Đây là cách xử lý **gốc rễ** vấn đề nhiễu mà đồ án gốc phải chế mạch khử rung riêng để đối phó.

- `PUL` / `DIR` dùng **cáp xoắn đôi có lưới chắn**, lưới nối đất **một đầu** phía tủ điện
- Cáp tín hiệu tách khỏi cáp động lực (pha động cơ, spindle) **tối thiểu 10 cm**; giao nhau thì
  cắt **vuông góc 90°**
- Cáp từ driver ra động cơ cũng phải có lưới chắn, nối PE tủ

### 7.4 PLC → BTS7960 (spindle)

BTS7960 dùng logic **3,3–5 V**, PLC ra **24 V** → **bắt buộc có board opto cách ly 24 V→5 V**.

| Đường | Đấu nối |
|---|---|
| Tốc độ | `Q0.6` (PWM4) → opto 24→5 V → `RPWM` |
| Đảo chiều (nhả dao) | `Q1.0` → relay chuyển PWM sang `LPWM` |
| Enable | `Q2.0` → opto → `R_EN` + `L_EN` |

**Phương án sạch hơn** nếu muốn tránh opto: dùng CPU **1215C** (có sẵn 2 ngõ ra analog) hoặc module
**SM 1232 AQ**, xuất **0–10 V** vào bộ điều tốc DC công nghiệp, đảo chiều bằng contactor.
Đắt hơn nhưng bỏ được cả BTS7960 lẫn board opto.

### 7.5 Driver → Động cơ bước

4 dây pha `A+` / `A−` / `B+` / `B−` tới NEMA17 (lưỡng cực), cáp có lưới chắn.

Hai thông số đặt bằng **DIP switch trên driver**, không phải bằng phần mềm:

| Thông số | Giá trị | Lý do |
|---|---|---|
| Dòng | ~1,4–1,6 A đỉnh | NEMA17 định mức 1,6 A |
| Vi bước | **1/16** → 3200 xung/vòng | Cho độ phân giải 2,5 µm/xung |

### 7.6 Tổng kết chuỗi tín hiệu

| Chặng | Phương tiện | Loại |
|---|---|---|
| Pi 4 → PLC | Ethernet RJ45, ISO-TCP:102, snap7 | **Giao thức số** |
| PLC → Driver bước | Xung/hướng 24 V + điện trở 2 kΩ, cáp lưới chắn | Tín hiệu cứng |
| PLC → BTS7960 | PWM 24 V → opto → 5 V | Tín hiệu cứng |
| Driver → NEMA17 | 4 dây pha có lưới chắn | Công suất |
| Probe / limit → PLC | Tiếp điểm khô 24 V vào DI | Tín hiệu cứng |

---

## Tài liệu liên quan

| File | Nội dung |
|---|---|
| [`thiet-bi-va-chuc-nang.md`](thiet-bi-va-chuc-nang.md) | Danh sách thiết bị và chức năng tổng thể |
| [`gerber-sang-nc.md`](gerber-sang-nc.md) | Sinh đường chạy dao từ file Gerber |
| [`kiem-tra-quang-hoc.md`](kiem-tra-quang-hoc.md) | Kiểm tra quang học tự động sau gia công |
| [`so-sanh-stm32-vs-s7.md`](so-sanh-stm32-vs-s7.md) | Đối chiếu bản gốc ↔ bản PLC |
