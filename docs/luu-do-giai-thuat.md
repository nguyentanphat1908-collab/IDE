# Lưu đồ giải thuật toàn hệ thống

Tập hợp các lưu đồ của máy gia công mạch PCB tự động bản PLC S7-1200.
Chi tiết từng phần nằm ở các tài liệu chuyên đề được dẫn trong mỗi mục.

## Đối chiếu với đồ án gốc

Đồ án gốc có 8 lưu đồ. Bảng dưới cho thấy cái nào giữ, cái nào thay, cái nào thêm mới:

| Lưu đồ đồ án gốc | Bản này | Quan hệ |
|---|---|---|
| Hình 6.2 — Sơ đồ khối tổng quát | §1 | Vẽ lại theo luồng Gerber |
| Hình 6.5 + 6.6 — Xử lý ảnh, xử lý dữ liệu | §2 | **Thay** bằng biên dịch Gerber → `.nc` |
| Hình 6.12 — Kết xuất chương trình vận hành | §2 | Giữ, vẽ lại |
| Hình 6.17 — Set home | §3 | Giữ, đổi sang `MC_Home` Mode 3 |
| Hình 6.14 — Leveling map | §4 | **Giữ nguyên giải thuật** |
| Hình 6.18 + 6.19 — Chạy XY, BLU_mapping | §5 | **Thay** bằng nội suy vận tốc tỉ lệ |
| — | §6 | **Mới** — thay dao tự động |
| — | §7 | **Mới** — bắt tay truyền dữ liệu |
| — | §8 | **Mới** — kiểm tra quang học |
| — | §9 | **Mới** — định vị và tối ưu phôi |
| — | §10 | **Mới** — máy trạng thái hệ thống |

---

## 1. Vận hành tổng thể

```mermaid
flowchart TD
    A([Bắt đầu]) --> B[/"Cắm USB: Gerber + Excellon"/]
    B --> C["Pi 4: pcb2gcode<br/>Gerber → file .nc"]
    C --> D[/"Lưu file .nc"/]
    D --> E["Chọn file và vùng gia công<br/>trên LCD cảm ứng"]
    E --> F["Set home: Z → X → Y"]
    F --> G["Định vị phôi bằng camera"]
    G --> H["Leveling map 66 điểm"]
    H --> I["Pi: phân tích G-code<br/>tuyến tính hóa cung tròn<br/>bù z, chia nhỏ đoạn dài"]
    I --> J["Truyền xuống DB_Buffer<br/>qua snap7"]
    J --> K["PLC thực thi quy trình"]
    K --> L{"Cần thay dao?"}
    L -- có --> M["Chuỗi ATC"]
    M --> K
    L -- không --> N{"Còn lệnh?"}
    N -- còn --> J
    N -- hết --> O["Kiểm tra quang học AOI"]
    O --> P[/"Bản đồ khuyết tật"/]
    P --> Q([Kết thúc])
```

## 2. Biên dịch Gerber → chương trình chạy máy

Thay hoàn toàn pipeline xử lý ảnh của đồ án gốc. Chi tiết: [`gerber-sang-nc.md`](gerber-sang-nc.md).

```mermaid
flowchart TD
    A([Nhận file Gerber]) --> B["pcb2gcode: offset bán kính dao<br/>sinh file .nc"]
    B --> C["Phân tích cú pháp G-code"]
    C --> D{"Có lệnh G02/G03<br/>cung tròn?"}
    D -- có --> E["Tuyến tính hóa thành dây cung<br/>sai số < độ phân giải trục"]
    E --> F
    D -- không --> F["Tra leveling map<br/>bù z cho từng điểm"]
    F --> G{"Đoạn dài hơn 5 mm?"}
    G -- có --> H["Chia nhỏ để z bám bề mặt"]
    H --> I
    G -- không --> I["Gom nhóm theo mũi khoan<br/>giảm số lần thay dao"]
    I --> J["Sắp thứ tự đường chạy<br/>nearest-neighbour"]
    J --> K["Áp phép quay theta + tịnh tiến<br/>theo vị trí phôi"]
    K --> L([Chuỗi lệnh sẵn sàng truyền])
```

## 3. Set home

```mermaid
flowchart TD
    A([Set home]) --> B["MC_Power: cấp cả 3 trục"]
    B --> C["MC_Home trục Z, Mode 3"]
    C --> D{"Z chạm công tắc gốc?"}
    D -- chưa --> C
    D -- rồi --> E["Z = 0, nâng về vị trí an toàn"]
    E --> F["MC_Home trục X"]
    F --> G{"X chạm gốc?"}
    G -- chưa --> F
    G -- rồi --> H["X = 0"]
    H --> I["MC_Home trục Y"]
    I --> J{"Y chạm gốc?"}
    J -- chưa --> I
    J -- rồi --> K["Y = 0"]
    K --> L["Mở khóa chương trình gia công"]
    L --> M([READY])
```

> **Thứ tự Z → X → Y không được đổi.** Z phải lên hết trước; nếu dao còn cắm trong phôi mà bàn máy
> chạy X/Y là gãy dao và xước bo.

## 4. Leveling map 66 điểm

Giữ nguyên giải thuật của đồ án gốc. Điểm khác duy nhất: dừng dao bằng **ngắt phần cứng**.

```mermaid
flowchart TD
    A([Leveling map]) --> B["i = 0"]
    B --> C["Di chuyển XY tới điểm thứ i<br/>trong ma trận 6 × 11"]
    C --> D["Tắt spindle, đóng relay Q1.1<br/>ngắt nhiễu tín hiệu probe"]
    D --> E["MC_MoveVelocity: hạ Z chậm"]
    E --> F{"Probe I0.3 chạm đồng?"}
    F -- chưa --> E
    F -- rồi --> G["NGẮT PHẦN CỨNG → MC_Halt<br/>đọc z_i, ghi DB_Status"]
    G --> H["Nâng Z về an toàn"]
    H --> I["i = i + 1"]
    I --> J{"i < 66?"}
    J -- còn --> C
    J -- hết --> K["Pi: ghép mỗi 3 điểm<br/>thành 1 phương trình mặt phẳng"]
    K --> L["Xây hàm chọn mặt phẳng<br/>phù hợp nhất cho x, y bất kỳ"]
    L --> M([Đối tượng trả z từ x, y])
```

> Quét probe trong `OB1` thì sai số Z bằng cả một chu kỳ quét — với dao phay PCB ăn sâu vài chục µm
> thì đó là hỏng mạch. Ngắt phần cứng cho độ trễ cỡ µs.

## 5. Nội suy đường thẳng 2 trục

Thay Bresenham Line và BLU_mapping của đồ án gốc.
Chi tiết: [`chuc-nang-plc.md`](chuc-nang-plc.md) §4.2.

```mermaid
flowchart TD
    A([FB_LinearMove]) --> B[/"Nhận x_đích, y_đích, v"/]
    B --> C["dx = x_đích − x_hiện<br/>dy = y_đích − y_hiện"]
    C --> D["L = căn bậc hai của dx² + dy²"]
    D --> E{"L = 0?"}
    E -- đúng --> Z([Done])
    E -- sai --> F["v_x = v × trị tuyệt đối dx / L<br/>v_y = v × trị tuyệt đối dy / L"]
    F --> G["MC_WriteParam:<br/>ghi v_x cho X, v_y cho Y"]
    G --> H["Phát ĐỒNG THỜI<br/>MC_MoveAbsolute cho X và Y"]
    H --> I{"Cả hai trục Done?"}
    I -- chưa --> I
    I -- rồi --> J["Cập nhật vị trí hiện tại"]
    J --> Z
```

> Hai trục xuất phát cùng lúc, vận tốc tỉ lệ với quãng đường thành phần nên **về đích cùng lúc**
> → quỹ đạo là đường thẳng. Đây là cách duy nhất tạo đường thẳng trên các trục PTO độc lập.

## 6. Thay dao tự động

```mermaid
flowchart TD
    A([FB_ToolChange]) --> B[/"Nhận số ổ dao đích"/]
    B --> C{"Dao đích = dao đang gắn?"}
    C -- đúng --> Z([Done])
    C -- sai --> D["Nâng Z về vị trí an toàn"]
    D --> E["Di chuyển tới ổ dao HIỆN TẠI"]
    E --> F["Hạ Z vào ổ"]
    F --> G["Q1.0 đảo chiều spindle<br/>PWM nhả dao trong T_nhả"]
    G --> H["Nâng Z thoát khỏi dao"]
    H --> I["Di chuyển tới ổ dao MỚI"]
    I --> J["Hạ Z vào dao"]
    J --> K["Spindle quay thuận<br/>PWM siết dao trong T_siết"]
    K --> L["Nâng Z"]
    L --> M["Cập nhật FB_ToolMagazine"]
    M --> Z
```

> `T_nhả` và `T_siết` **phải thực nghiệm lại ở 24 V**. Giá trị 200/200 ms và 250/250 ms của đồ án
> gốc đo ở 12 V nên không còn đúng.

## 7. Bắt tay truyền dữ liệu

Giữ triết lý *"truyền không nhanh, nhưng phải đủ"* của đồ án gốc, đổi từ đếm **từng lệnh** sang
đếm **theo ô ring buffer**.

```mermaid
flowchart TD
    A([Truyền chương trình]) --> B["Reset int_sys và int_cnc"]
    B --> C["Nạp 32 đoạn đầu vào DB_Buffer"]
    C --> D["int_sys = int_sys + 1"]
    D --> E{"int_cnc đã khớp?"}
    E -- rồi --> F["Nạp tiếp vào các ô đã trống"]
    E -- chưa --> G{"Quá thời gian chờ?"}
    G -- chưa --> E
    G -- rồi --> H["GỬI LẠI khối cũ"]
    H --> E
    F --> I{"Còn lệnh trong file?"}
    I -- còn --> D
    I -- hết --> J["Chờ PLC chạy hết buffer"]
    J --> K([Xong])
```

> Độ trễ Ethernet 5–20 ms mỗi lượt khiến bắt tay từng lệnh chỉ đạt 50–200 lệnh/s — nghẽn với đường
> mạch nhiều đoạn ngắn. Ring buffer giải quyết việc này.

## 8. Kiểm tra quang học sau gia công

Chi tiết: [`kiem-tra-quang-hoc.md`](kiem-tra-quang-hoc.md).

```mermaid
flowchart TD
    A([Gia công xong]) --> B["Tắt spindle M05<br/>nâng Z về chiều cao lấy nét"]
    B --> C["k = 1"]
    C --> D["PLC: di chuyển tới tile thứ k"]
    D --> E["Chờ Done + ổn định 0,5 s"]
    E --> F["Pi: chụp ảnh tile k"]
    F --> G["Khử méo ống kính<br/>phân đoạn HSV → mặt nạ đồng"]
    G --> H["XOR với mặt nạ Gerber<br/>đăng ký theo tọa độ máy"]
    H --> I["Lọc vùng nhỏ hơn A_min<br/>lưu vùng ứng viên"]
    I --> J["Giải phóng ảnh tile k"]
    J --> K["k = k + 1"]
    K --> L{"k nhỏ hơn hoặc bằng 16?"}
    L -- còn --> D
    L -- hết --> M["YOLO phân loại<br/>CHỈ trên vùng ứng viên"]
    M --> N["NMS, quy đổi tọa độ về mm"]
    N --> O[/"Bản đồ khuyết tật D1–D8"/]
    O --> P([Hiển thị lên LCD])
```

> Bước **giải phóng ảnh ngay sau mỗi tile** là bắt buộc: 16 tile là 0,59 GB dữ liệu thô, giữ hết
> trong RAM là tràn.

## 9. Định vị phôi và tối ưu vị trí đặt

Chi tiết: [`dinh-vi-va-toi-uu-phoi.md`](dinh-vi-va-toi-uu-phoi.md).

```mermaid
flowchart TD
    A([Đơn hàng mới]) --> B["Đặt phôi vị trí bất kỳ"]
    B --> C["Quét 16 tile, binning 4×"]
    C --> D{"Tìm thấy phôi?"}
    D -- không --> E[/"Báo: không thấy phôi"/]
    E --> Z([Dừng])
    D -- có --> F["Phân đoạn 3 lớp<br/>đóng hình thái gộp vùng đã dùng"]
    F --> G["Đo tinh biên, khớp minAreaRect<br/>→ W, H, theta, gốc phôi"]
    G --> H{"theta vượt<br/>giới hạn hành trình?"}
    H -- có --> I[/"Báo: đặt lại phôi thẳng hơn"/]
    I --> Z
    H -- không --> J["Dựng bản đồ chiếm dụng<br/>lưới 1 mm, hệ tọa độ phôi"]
    J --> K["Duyệt điểm góc lõm × 2 hướng xoay"]
    K --> L["Loại vị trí sinh vách mỏng"]
    L --> M{"Còn vị trí nào?"}
    M -- không --> N[/"Báo: phôi không đủ chỗ"/]
    N --> Z
    M -- có --> O["Chấm điểm: mảnh tự do lớn nhất"]
    O --> P["Chọn điểm cao nhất"]
    P --> Q[/"Mô phỏng, người dùng xác nhận"/]
    Q --> R([Áp phép biến đổi cho .nc])
```

## 10. Máy trạng thái hệ thống

```mermaid
stateDiagram-v2
    [*] --> IDLE
    IDLE --> HOMING: Start
    HOMING --> LOCATING: về gốc xong
    LOCATING --> LEVELING: định vị phôi xong
    LEVELING --> READY: đủ 66 điểm
    READY --> RUNNING: nhận lệnh gia công
    RUNNING --> RUNNING: lệnh kế tiếp
    RUNNING --> PAUSED: Stop
    PAUSED --> RUNNING: Start
    RUNNING --> INSPECTING: hết chương trình
    INSPECTING --> READY: AOI xong
    RUNNING --> ERROR: limit / alarm driver / watchdog
    LOCATING --> ERROR: không thấy phôi / theta vượt hạn
    LEVELING --> ERROR: probe lỗi
    ERROR --> IDLE: Reset
```

---

## Tài liệu liên quan

| File | Nội dung |
|---|---|
| [`thiet-bi-va-chuc-nang.md`](thiet-bi-va-chuc-nang.md) | Danh sách thiết bị và chức năng tổng thể |
| [`chuc-nang-plc.md`](chuc-nang-plc.md) | Đặc tả PLC: I/O, khối hàm, sơ đồ nguyên lý |
| [`gerber-sang-nc.md`](gerber-sang-nc.md) | Sinh đường chạy dao từ file Gerber |
| [`kiem-tra-quang-hoc.md`](kiem-tra-quang-hoc.md) | Kiểm tra quang học tự động sau gia công |
| [`dinh-vi-va-toi-uu-phoi.md`](dinh-vi-va-toi-uu-phoi.md) | Định vị phôi và tối ưu tận dụng phôi thừa |
