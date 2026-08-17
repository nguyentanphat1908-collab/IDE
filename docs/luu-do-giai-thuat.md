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

## Quy ước ký hiệu

Các lưu đồ dưới đây dùng bộ ký hiệu chuẩn:

| Ký hiệu | Hình dạng | Ý nghĩa |
|---|---|---|
| `([...])` | Bo tròn | Bắt đầu / Kết thúc |
| `[/.../]` | Bình hành | Nhập / Xuất dữ liệu |
| `[...]` | Chữ nhật | Xử lý |
| `{...}` | Thoi | Quyết định — hai nhánh **Đúng** / **Sai** |
| `(( ))` | Tròn nhỏ | Điểm nối các nhánh |

---

## 1. Vận hành tổng thể

```mermaid
flowchart TD
    A([Bắt đầu]):::term --> B[/"Cắm USB: Gerber + Excellon"/]:::io
    B --> C["Pi 4: pcb2gcode<br/>Gerber → file .nc"]:::proc
    C --> D[/"Lưu file .nc"/]:::io
    D --> E["Chọn file và vùng gia công<br/>trên LCD cảm ứng"]:::proc
    E --> F["Set home: Z → X → Y"]:::proc
    F --> G["Định vị phôi bằng camera"]:::proc
    G --> H["Leveling map 66 điểm"]:::proc
    H --> I["Pi: phân tích G-code<br/>tuyến tính hóa cung tròn<br/>bù z, chia nhỏ đoạn dài"]:::proc
    I --> J["Truyền xuống DB_Buffer<br/>qua snap7"]:::proc
    J --> K["PLC thực thi quy trình"]:::proc
    K --> L{"Cần thay dao?"}:::dec
    L -- Đúng --> M["Chuỗi ATC"]:::proc
    M --> K
    L -- Sai --> N{"Còn lệnh?"}:::dec
    N -- Đúng --> J
    N -- Sai --> O["Kiểm tra quang học AOI"]:::proc
    O --> P[/"Bản đồ khuyết tật"/]:::io
    P --> Q([Kết thúc]):::term
    classDef term fill:#BDD7EE,stroke:#2E75B6,stroke-width:2px,color:#000
    classDef proc fill:#DEEBF7,stroke:#5B9BD5,stroke-width:2px,color:#000
    classDef io   fill:#DEEBF7,stroke:#5B9BD5,stroke-width:2px,color:#000
    classDef dec  fill:#9DC3E6,stroke:#2E75B6,stroke-width:2px,color:#000
    classDef conn fill:#9DC3E6,stroke:#2E75B6,stroke-width:2px,color:#000
    linkStyle default stroke:#5B9BD5,stroke-width:2px
```

## 2. Biên dịch Gerber → chương trình chạy máy

Thay hoàn toàn pipeline xử lý ảnh của đồ án gốc. Chi tiết: [`gerber-sang-nc.md`](gerber-sang-nc.md).

```mermaid
flowchart TD
    A([Nhận file Gerber]):::term --> B["pcb2gcode: offset bán kính dao<br/>sinh file .nc"]:::proc
    B --> C["Phân tích cú pháp G-code"]:::proc
    C --> D{"Có lệnh G02/G03<br/>cung tròn?"}:::dec
    D -- Đúng --> E["Tuyến tính hóa thành dây cung<br/>sai số < độ phân giải trục"]:::proc
    E --> F
    D -- Sai --> F["Tra leveling map<br/>bù z cho từng điểm"]:::proc
    F --> G{"Đoạn dài hơn 5 mm?"}:::dec
    G -- Đúng --> H["Chia nhỏ để z bám bề mặt"]:::proc
    H --> I
    G -- Sai --> I["Gom nhóm theo mũi khoan<br/>giảm số lần thay dao"]:::proc
    I --> J["Sắp thứ tự đường chạy<br/>nearest-neighbour"]:::proc
    J --> K["Áp phép quay theta + tịnh tiến<br/>theo vị trí phôi"]:::proc
    K --> L([Chuỗi lệnh sẵn sàng truyền]):::term
    classDef term fill:#BDD7EE,stroke:#2E75B6,stroke-width:2px,color:#000
    classDef proc fill:#DEEBF7,stroke:#5B9BD5,stroke-width:2px,color:#000
    classDef io   fill:#DEEBF7,stroke:#5B9BD5,stroke-width:2px,color:#000
    classDef dec  fill:#9DC3E6,stroke:#2E75B6,stroke-width:2px,color:#000
    classDef conn fill:#9DC3E6,stroke:#2E75B6,stroke-width:2px,color:#000
    linkStyle default stroke:#5B9BD5,stroke-width:2px
```

## 3. Set home

```mermaid
flowchart TD
    A([Set home]):::term --> B["MC_Power: cấp cả 3 trục"]:::proc
    B --> C["MC_Home trục Z, Mode 3"]:::proc
    C --> D{"Z chạm công tắc gốc?"}:::dec
    D -- Sai --> C
    D -- Đúng --> E["Z = 0, nâng về vị trí an toàn"]:::proc
    E --> F["MC_Home trục X"]:::proc
    F --> G{"X chạm gốc?"}:::dec
    G -- Sai --> F
    G -- Đúng --> H["X = 0"]:::proc
    H --> I["MC_Home trục Y"]:::proc
    I --> J{"Y chạm gốc?"}:::dec
    J -- Sai --> I
    J -- Đúng --> K["Y = 0"]:::proc
    K --> L["Mở khóa chương trình gia công"]:::proc
    L --> M([READY]):::term
    classDef term fill:#BDD7EE,stroke:#2E75B6,stroke-width:2px,color:#000
    classDef proc fill:#DEEBF7,stroke:#5B9BD5,stroke-width:2px,color:#000
    classDef io   fill:#DEEBF7,stroke:#5B9BD5,stroke-width:2px,color:#000
    classDef dec  fill:#9DC3E6,stroke:#2E75B6,stroke-width:2px,color:#000
    classDef conn fill:#9DC3E6,stroke:#2E75B6,stroke-width:2px,color:#000
    linkStyle default stroke:#5B9BD5,stroke-width:2px
```

> **Thứ tự Z → X → Y không được đổi.** Z phải lên hết trước; nếu dao còn cắm trong phôi mà bàn máy
> chạy X/Y là gãy dao và xước bo.

## 4. Leveling map 66 điểm

Giữ nguyên giải thuật của đồ án gốc. Điểm khác duy nhất: dừng dao bằng **ngắt phần cứng**.

```mermaid
flowchart TD
    A([Leveling map]):::term --> B["i = 0"]:::proc
    B --> C["Di chuyển XY tới điểm thứ i<br/>trong ma trận 6 × 11"]:::proc
    C --> D["Tắt spindle, đóng relay Q1.1<br/>ngắt nhiễu tín hiệu probe"]:::proc
    D --> E["MC_MoveVelocity: hạ Z chậm"]:::proc
    E --> F{"Probe I0.3 chạm đồng?"}:::dec
    F -- Sai --> E
    F -- Đúng --> G["NGẮT PHẦN CỨNG → MC_Halt<br/>đọc z_i, ghi DB_Status"]:::proc
    G --> H["Nâng Z về an toàn"]:::proc
    H --> I["i = i + 1"]:::proc
    I --> J{"i < 66?"}:::dec
    J -- Đúng --> C
    J -- Sai --> K["Pi: ghép mỗi 3 điểm<br/>thành 1 phương trình mặt phẳng"]:::proc
    K --> L["Xây hàm chọn mặt phẳng<br/>phù hợp nhất cho x, y bất kỳ"]:::proc
    L --> M([Đối tượng trả z từ x, y]):::term
    classDef term fill:#BDD7EE,stroke:#2E75B6,stroke-width:2px,color:#000
    classDef proc fill:#DEEBF7,stroke:#5B9BD5,stroke-width:2px,color:#000
    classDef io   fill:#DEEBF7,stroke:#5B9BD5,stroke-width:2px,color:#000
    classDef dec  fill:#9DC3E6,stroke:#2E75B6,stroke-width:2px,color:#000
    classDef conn fill:#9DC3E6,stroke:#2E75B6,stroke-width:2px,color:#000
    linkStyle default stroke:#5B9BD5,stroke-width:2px
```

> Quét probe trong `OB1` thì sai số Z bằng cả một chu kỳ quét — với dao phay PCB ăn sâu vài chục µm
> thì đó là hỏng mạch. Ngắt phần cứng cho độ trễ cỡ µs.

## 5. Nội suy đường thẳng 2 trục

Thay Bresenham Line và BLU_mapping của đồ án gốc.
Chi tiết: [`chuc-nang-plc.md`](chuc-nang-plc.md) §4.2.

```mermaid
flowchart TD
    A([FB_LinearMove]):::term --> B[/"Nhận x_đích, y_đích, v"/]:::io
    B --> C["dx = x_đích − x_hiện<br/>dy = y_đích − y_hiện"]:::proc
    C --> D["L = căn bậc hai của dx² + dy²"]:::proc
    D --> E{"L = 0?"}:::dec
    E -- Đúng --> Z([Done]):::term
    E -- Sai --> F["v_x = v × trị tuyệt đối dx / L<br/>v_y = v × trị tuyệt đối dy / L"]:::proc
    F --> G["MC_WriteParam:<br/>ghi v_x cho X, v_y cho Y"]:::proc
    G --> H["Phát ĐỒNG THỜI<br/>MC_MoveAbsolute cho X và Y"]:::proc
    H --> I{"Cả hai trục Done?"}:::dec
    I -- Sai --> I
    I -- Đúng --> J["Cập nhật vị trí hiện tại"]:::proc
    J --> Z
    classDef term fill:#BDD7EE,stroke:#2E75B6,stroke-width:2px,color:#000
    classDef proc fill:#DEEBF7,stroke:#5B9BD5,stroke-width:2px,color:#000
    classDef io   fill:#DEEBF7,stroke:#5B9BD5,stroke-width:2px,color:#000
    classDef dec  fill:#9DC3E6,stroke:#2E75B6,stroke-width:2px,color:#000
    classDef conn fill:#9DC3E6,stroke:#2E75B6,stroke-width:2px,color:#000
    linkStyle default stroke:#5B9BD5,stroke-width:2px
```

> Hai trục xuất phát cùng lúc, vận tốc tỉ lệ với quãng đường thành phần nên **về đích cùng lúc**
> → quỹ đạo là đường thẳng. Đây là cách duy nhất tạo đường thẳng trên các trục PTO độc lập.

## 6. Thay dao tự động

```mermaid
flowchart TD
    A([FB_ToolChange]):::term --> B[/"Nhận số ổ dao đích"/]:::io
    B --> C{"Dao đích = dao đang gắn?"}:::dec
    C -- Đúng --> Z([Done]):::term
    C -- Sai --> D["Nâng Z về vị trí an toàn"]:::proc
    D --> E["Di chuyển tới ổ dao HIỆN TẠI"]:::proc
    E --> F["Hạ Z vào ổ"]:::proc
    F --> G["Q1.0 đảo chiều spindle<br/>PWM nhả dao trong T_nhả"]:::proc
    G --> H["Nâng Z thoát khỏi dao"]:::proc
    H --> I["Di chuyển tới ổ dao MỚI"]:::proc
    I --> J["Hạ Z vào dao"]:::proc
    J --> K["Spindle quay thuận<br/>PWM siết dao trong T_siết"]:::proc
    K --> L["Nâng Z"]:::proc
    L --> M["Cập nhật FB_ToolMagazine"]:::proc
    M --> Z
    classDef term fill:#BDD7EE,stroke:#2E75B6,stroke-width:2px,color:#000
    classDef proc fill:#DEEBF7,stroke:#5B9BD5,stroke-width:2px,color:#000
    classDef io   fill:#DEEBF7,stroke:#5B9BD5,stroke-width:2px,color:#000
    classDef dec  fill:#9DC3E6,stroke:#2E75B6,stroke-width:2px,color:#000
    classDef conn fill:#9DC3E6,stroke:#2E75B6,stroke-width:2px,color:#000
    linkStyle default stroke:#5B9BD5,stroke-width:2px
```

> `T_nhả` và `T_siết` **phải thực nghiệm lại ở 24 V**. Giá trị 200/200 ms và 250/250 ms của đồ án
> gốc đo ở 12 V nên không còn đúng.

## 7. Bắt tay truyền dữ liệu

Giữ triết lý *"truyền không nhanh, nhưng phải đủ"* của đồ án gốc, đổi từ đếm **từng lệnh** sang
đếm **theo ô ring buffer**.

```mermaid
flowchart TD
    A([Truyền chương trình]):::term --> B["Reset int_sys và int_cnc"]:::proc
    B --> C["Nạp 32 đoạn đầu vào DB_Buffer"]:::proc
    C --> D["int_sys = int_sys + 1"]:::proc
    D --> E{"int_cnc đã khớp?"}:::dec
    E -- Đúng --> F["Nạp tiếp vào các ô đã trống"]:::proc
    E -- Sai --> G{"Quá thời gian chờ?"}:::dec
    G -- Sai --> E
    G -- Đúng --> H["GỬI LẠI khối cũ"]:::proc
    H --> E
    F --> I{"Còn lệnh trong file?"}:::dec
    I -- Đúng --> D
    I -- Sai --> J["Chờ PLC chạy hết buffer"]:::proc
    J --> K([Xong]):::term
    classDef term fill:#BDD7EE,stroke:#2E75B6,stroke-width:2px,color:#000
    classDef proc fill:#DEEBF7,stroke:#5B9BD5,stroke-width:2px,color:#000
    classDef io   fill:#DEEBF7,stroke:#5B9BD5,stroke-width:2px,color:#000
    classDef dec  fill:#9DC3E6,stroke:#2E75B6,stroke-width:2px,color:#000
    classDef conn fill:#9DC3E6,stroke:#2E75B6,stroke-width:2px,color:#000
    linkStyle default stroke:#5B9BD5,stroke-width:2px
```

> Độ trễ Ethernet 5–20 ms mỗi lượt khiến bắt tay từng lệnh chỉ đạt 50–200 lệnh/s — nghẽn với đường
> mạch nhiều đoạn ngắn. Ring buffer giải quyết việc này.

## 8. Kiểm tra quang học sau gia công

Chi tiết: [`kiem-tra-quang-hoc.md`](kiem-tra-quang-hoc.md).

```mermaid
flowchart TD
    A([Gia công xong]):::term --> B["Tắt spindle M05<br/>nâng Z về chiều cao lấy nét"]:::proc
    B --> C["k = 1"]:::proc
    C --> D["PLC: di chuyển tới tile thứ k"]:::proc
    D --> E["Chờ Done + ổn định 0,5 s"]:::proc
    E --> F["Pi: chụp ảnh tile k"]:::proc
    F --> G["Khử méo ống kính<br/>phân đoạn HSV → mặt nạ đồng"]:::proc
    G --> H["XOR với mặt nạ Gerber<br/>đăng ký theo tọa độ máy"]:::proc
    H --> I["Lọc vùng nhỏ hơn A_min<br/>lưu vùng ứng viên"]:::proc
    I --> J["Giải phóng ảnh tile k"]:::proc
    J --> K["k = k + 1"]:::proc
    K --> L{"k nhỏ hơn hoặc bằng 16?"}:::dec
    L -- Đúng --> D
    L -- Sai --> M["YOLO phân loại<br/>CHỈ trên vùng ứng viên"]:::proc
    M --> N["NMS, quy đổi tọa độ về mm"]:::proc
    N --> O[/"Bản đồ khuyết tật D1–D8"/]:::io
    O --> P([Hiển thị lên LCD]):::term
    classDef term fill:#BDD7EE,stroke:#2E75B6,stroke-width:2px,color:#000
    classDef proc fill:#DEEBF7,stroke:#5B9BD5,stroke-width:2px,color:#000
    classDef io   fill:#DEEBF7,stroke:#5B9BD5,stroke-width:2px,color:#000
    classDef dec  fill:#9DC3E6,stroke:#2E75B6,stroke-width:2px,color:#000
    classDef conn fill:#9DC3E6,stroke:#2E75B6,stroke-width:2px,color:#000
    linkStyle default stroke:#5B9BD5,stroke-width:2px
```

> Bước **giải phóng ảnh ngay sau mỗi tile** là bắt buộc: 16 tile là 0,39 GB dữ liệu thô, giữ hết
> trong RAM là tràn.

## 9. Định vị phôi và tối ưu vị trí đặt

Chi tiết: [`dinh-vi-va-toi-uu-phoi.md`](dinh-vi-va-toi-uu-phoi.md).

```mermaid
flowchart TD
    A([Đơn hàng mới]):::term --> B["Đặt phôi vị trí bất kỳ"]:::proc
    B --> C["Quét 16 tile, binning 4×"]:::proc
    C --> D{"Tìm thấy phôi?"}:::dec
    D -- Sai --> E[/"Báo: không thấy phôi"/]:::io
    E --> Z([Dừng]):::term
    D -- Đúng --> F["Phân đoạn 3 lớp<br/>đóng hình thái gộp vùng đã dùng"]:::proc
    F --> G["Đo tinh biên, khớp minAreaRect<br/>→ W, H, theta, gốc phôi"]:::proc
    G --> H{"theta vượt<br/>giới hạn hành trình?"}:::dec
    H -- Đúng --> I[/"Báo: đặt lại phôi thẳng hơn"/]:::io
    I --> Z
    H -- Sai --> J["Dựng bản đồ chiếm dụng<br/>lưới 1 mm, hệ tọa độ phôi"]:::proc
    J --> K["Duyệt điểm góc lõm × 2 hướng xoay"]:::proc
    K --> L["Loại vị trí sinh vách mỏng"]:::proc
    L --> M{"Còn vị trí nào?"}:::dec
    M -- Sai --> N[/"Báo: phôi không đủ chỗ"/]:::io
    N --> Z
    M -- Đúng --> O["Chấm điểm: mảnh tự do lớn nhất"]:::proc
    O --> P["Chọn điểm cao nhất"]:::proc
    P --> Q[/"Mô phỏng, người dùng xác nhận"/]:::io
    Q --> R([Áp phép biến đổi cho .nc]):::term
    classDef term fill:#BDD7EE,stroke:#2E75B6,stroke-width:2px,color:#000
    classDef proc fill:#DEEBF7,stroke:#5B9BD5,stroke-width:2px,color:#000
    classDef io   fill:#DEEBF7,stroke:#5B9BD5,stroke-width:2px,color:#000
    classDef dec  fill:#9DC3E6,stroke:#2E75B6,stroke-width:2px,color:#000
    classDef conn fill:#9DC3E6,stroke:#2E75B6,stroke-width:2px,color:#000
    linkStyle default stroke:#5B9BD5,stroke-width:2px
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
    classDef proc fill:#DEEBF7,stroke:#5B9BD5,stroke-width:2px,color:#000
    class IDLE,HOMING,LOCATING,LEVELING,READY,RUNNING,INSPECTING,PAUSED,ERROR proc
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
