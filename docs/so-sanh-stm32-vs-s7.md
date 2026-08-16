# Đối chiếu: bản gốc (STM32) ↔ bản mới (PLC S7-1200)

Tài liệu này nêu thẳng **cái mất và cái được** khi chuyển kiến trúc điều khiển, để người đọc tự
đánh giá xem đánh đổi có xứng đáng với mục đích của mình không.

---

## 1. Bảng đối chiếu tổng thể

| Hạng mục | Bản gốc (đồ án 2023) | Bản mới (PLC) |
|---|---|---|
| **Bộ điều khiển** | STM32F103C8T6 (Flash 64 KB, SRAM 20 KB) | S7-1200 CPU 1214C DC/DC/DC + SM 1223 |
| **Máy tính nhúng** | Raspberry Pi Zero 2W (512 MB, **không có Ethernet**) | Raspberry Pi 4 (4 GB, **Gigabit Ethernet**) |
| **Kết nối Master↔Slave** | UART | **Ethernet — ISO-TCP:102, python-snap7** |
| **Driver bước** | 3× DRV8825 (logic 3,3–5 V, 2,5 A/pha) | 3× Leadshine DM542 / TB6600 (nhận 24 V + điện trở 2 kΩ) |
| **Chống nhiễu** | Mạch khử rung tự thiết kế | Opto sẵn trong driver + cáp lưới chắn + đi dây đúng chuẩn |
| **Phát xung** | Bit-bang bằng firmware | **PTO phần cứng, 100 kHz** |
| **Nội suy quỹ đạo** | **Bresenham Line + BLU_mapping 16 hướng** | Vận tốc tỉ lệ trên 2 trục PTO độc lập |
| **Định dạng Acode** | `bit x,y` — **từng pixel** | `line x,y,f` — **polyline** |
| **Điện áp hệ thống** | 12 VDC (nguồn 20 A) | **24 VDC** (nguồn 15 A + PM1207 riêng cho PLC) |
| **Ngôn ngữ / công cụ** | C++ trên PlatformIO (Firmware_ADL) | SCL + LAD trên **TIA Portal** |
| **Nạp chương trình** | USB / SWD, miễn phí | TIA Portal — **cần license** |
| **Bo mạch** | Tự thiết kế dạng module, nối dây bus | Tủ điện, DIN rail, terminal chuẩn công nghiệp |
| **An toàn** | Không có mạch dừng khẩn độc lập | **E-stop + relay an toàn đấu cứng**, watchdog, giám sát driver |
| **Chi phí bộ điều khiển** | Rất thấp (~vài trăm nghìn VNĐ) | **Cao hơn nhiều lần** (CPU + SM + license) |

---

## 2. Cái mất

### 2.1 Mất nội suy ở mức pixel — mất luôn giá trị học thuật lớn nhất

Đây là điểm quan trọng nhất, cần nói thẳng.

Giá trị lớn nhất của đồ án gốc **không phải** cái máy, mà là việc nhóm tác giả **từ chối dùng mã
nguồn mở có sẵn** (GRBL, Mach3, Aspire) và tự viết lại toàn bộ: tự thiết kế định dạng Acode, tự
hiện thực Bresenham, tự mở rộng thành BLU_mapping 16 hướng, tự xây Framework_ADL và Firmware_ADL.
Đồ án nêu rõ lý do: các nghiên cứu dùng mã nguồn mở *"không giúp chúng ta hiểu rõ tường tận về cách
thức vận hành, tổ chức và xử lí dữ liệu"*.

Chuyển sang PLC là **quay lại đúng chỗ mà đồ án cố tránh** — dùng thư viện motion đóng của Siemens.
`MC_MoveAbsolute` là hộp đen; bạn không còn biết bên trong nó nội suy thế nào.

**Bresenham Line và BLU_mapping bị loại bỏ hoàn toàn.** Không có cách nào giữ lại chúng trên PLC.

> **Nếu đây là đồ án nộp trường**, hội đồng nhiều khả năng sẽ hỏi đúng điểm này. Cần chuẩn bị lập
> luận: chuyển sang PLC là bài toán **công nghiệp hóa một nguyên mẫu**, khác mục tiêu với bài toán
> **làm chủ công nghệ** của đồ án gốc. Hai mục tiêu đều chính đáng nhưng không thể đạt đồng thời.

### 2.2 Chi phí tăng đáng kể

Đồ án gốc đặt mục tiêu *"chi phí thấp để học sinh, sinh viên tiếp cận được"* — đó là lý do tồn tại
của đề tài. CPU S7-1200 cộng SM 1223 cộng license TIA Portal đắt hơn STM32 nhiều lần, làm suy yếu
chính lý do đó.

### 2.3 Phải sửa cả phần mềm PC, không chỉ PLC

Đổi từ pixel sang polyline nghĩa là **khâu xử lý ảnh phải viết lại**, chứ không phải chỉ thay bộ
điều khiển. Khối lượng công việc lớn hơn nhiều so với cảm nhận ban đầu.

### 2.4 Không có nội suy cung tròn

Cách nội suy bằng vận tốc tỉ lệ chỉ dựng được **đường thẳng**. Với gia công PCB thì đủ, nhưng nếu
sau này muốn mở rộng sang gia công biên dạng cong thật sự thì phải lên S7-1500T (có kinematics)
với chi phí cao hơn nữa.

### 2.5 Thông số thực nghiệm ATC hết hiệu lực

Bảng 4.1 và 4.2 của đồ án — kết quả của một quá trình thực nghiệm nghiêm túc (5 lần mỗi bộ thông
số, 17 trường hợp) — **đo ở 12 V nên không dùng được ở 24 V**. Phải làm lại từ đầu.

---

## 3. Cái được

### 3.1 Độ tin cậy công nghiệp

PLC được thiết kế để chạy liên tục nhiều năm trong môi trường nhiễu, nhiệt độ dao động, rung động.
STM32 trên bo tự hàn thì không có đảm bảo đó. Đồ án gốc tự nhận xét *"phần đi dây chưa thật sự
đẹp mắt"* — với PLC và tủ điện chuẩn thì vấn đề này biến mất.

### 3.2 An toàn — khác biệt lớn nhất về chất

Bản gốc **không có mạch dừng khẩn độc lập**. Bản mới có:

- E-stop đấu cứng qua relay an toàn, cắt thẳng nguồn động lực, **không phụ thuộc PLC**
- Watchdog mất kết nối → tự dừng
- Giám sát chân alarm driver → dừng khi mất bước hoặc quá nhiệt
- Giới hạn mềm trong Technology Object
- Khóa gia công cho đến khi về gốc xong

Với máy có dao quay 12000 v/ph, đây không phải chi tiết phụ.

### 3.3 Chẩn đoán và bảo trì

TIA Portal có online monitoring, force table, trace — xem được giá trị mọi biến trong lúc máy chạy.
Debug firmware STM32 trên máy đang gia công khó hơn nhiều.

Linh kiện Siemens có sẵn ở mọi nơi, thay thế trong ngày. Bo mạch tự thiết kế hỏng thì phải tự sửa.

### 3.4 Thư viện motion đã kiểm chứng

`MC_Home`, `MC_MoveAbsolute` và các Technology Object đã được kiểm chứng qua hàng triệu máy. Tự
viết homing và điều khiển vị trí thì phải tự tìm ra hết các trường hợp biên.

### 3.5 Ethernet thay UART

Băng thông cao hơn, khoảng cách xa hơn, có thể giám sát từ xa, và bỏ được vấn đề nhiễu trên đường
UART dài trong tủ điện.

### 3.6 Giá trị đào tạo khác — không kém, chỉ khác

Đồ án gốc dạy: kiến trúc firmware, giải thuật đồ họa, lập trình nhúng.
Bản PLC dạy: TIA Portal, Technology Object, truyền thông công nghiệp, thiết kế tủ điện, an toàn máy.

Kỹ năng thứ hai là thứ nhà tuyển dụng trong ngành tự động hóa tìm kiếm trực tiếp.

---

## 4. Cái giữ nguyên

| Hạng mục | Ghi chú |
|---|---|
| **Toàn bộ phần cơ khí** | Vitme T8, thanh trượt tròn, gối bi KFL08, ổ dao 6 ổ, khung máy — không đổi một chi tiết |
| **Động cơ** | 3× NEMA17 và RS775, chỉ đổi điện áp vận hành sang 24 V |
| **Leveling map 66 điểm 6×11** | Giữ nguyên giải thuật, chuyển sang chạy trên Pi |
| **Triết lý bắt tay** | *"Truyền không nhanh, nhưng phải đủ"* — cặp đếm `int_sys`/`int_cnc` chuyển thành word trong DB |
| **Kiến trúc 3 tầng** | Unit → Int → Main của Firmware_ADL ánh xạ tự nhiên sang FB/FC của TIA Portal |
| **Framework_ADL** | Vẫn dùng cho phần mềm PC và giao diện trên Pi |
| **Luồng 7 tab phần mềm PC** | Không đổi |
| **Tư tưởng xử lý ảnh** | Canny, tìm tâm lỗ khoan, tô đen lỗ khoan — giữ nguyên, chỉ thay khâu cuối |

Đáng chú ý: **kiến trúc phần mềm của đồ án gốc tốt đến mức chuyển sang PLC vẫn dùng lại được**.
Đó là lời khen thực chất cho thiết kế Firmware_ADL — một firmware hướng đối tượng phân tầng rõ ràng
thì việc đổi nền tảng phần cứng không phá vỡ cấu trúc.

---

## 5. Khi nào nên chọn phương án nào

| Nếu mục tiêu là… | Nên chọn |
|---|---|
| Làm chủ công nghệ, hiểu tường tận giải thuật CNC | **Bản gốc STM32** |
| Đồ án tốt nghiệp ngành Cơ điện tử nhấn mạnh lập trình nhúng | **Bản gốc STM32** |
| Chi phí thấp nhất để nhiều sinh viên tiếp cận | **Bản gốc STM32** |
| Máy chạy thật, nhiều người dùng chung, cần an toàn | **Bản PLC** |
| Học kỹ năng tự động hóa công nghiệp (TIA Portal, PROFINET) | **Bản PLC** |
| Đưa vào xưởng hoặc phòng thí nghiệm vận hành lâu dài | **Bản PLC** |

---

## Tài liệu liên quan

| File | Nội dung |
|---|---|
| [`thiet-bi-va-chuc-nang.md`](thiet-bi-va-chuc-nang.md) | Danh sách thiết bị và chức năng tổng thể |
| [`chuc-nang-plc.md`](chuc-nang-plc.md) | Đặc tả PLC: I/O, khối hàm, DB, an toàn, đấu dây |
| [`xu-ly-anh.md`](xu-ly-anh.md) | Pipeline PDF → Acode polyline |
