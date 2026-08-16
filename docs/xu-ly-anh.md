# Pipeline xử lý ảnh: PDF → Acode polyline

Đặc tả phần mềm xử lý ảnh chạy trên PC (xuất file) và Raspberry Pi 4 (biên dịch).
Xem `thiet-bi-va-chuc-nang.md` cho bối cảnh tổng thể.

---

## Vì sao phải viết lại khâu cuối

Đồ án gốc xuất **từng pixel** (`bit x,y`) vì STM32 tự bit-bang xung nên chạy được giải thuật
Bresenham. Với PLC, cách đó không dùng được vì **hai lý do độc lập nhau — mỗi lý do đều đủ để loại**:

| # | Lý do | Hệ quả |
|---|---|---|
| 1 | PTO của S7-1200 là các trục **độc lập**, không có nội suy đường thẳng | Phải gửi từng đoạn thẳng, không thể gửi từng điểm |
| 2 | Độ trễ Ethernet **5–20 ms/lệnh** → trần 50–200 lệnh/s | Gửi hàng nghìn pixel là bất khả thi |

**Toàn bộ tư tưởng xử lý ảnh của đồ án gốc được giữ nguyên** — chỉ thay khâu cuối từ pixel sang
**polyline** (contour tracing + rút gọn Douglas–Peucker).

## Thư viện

| Thư viện | Vai trò | Ghi chú |
|---|---|---|
| **PyMuPDF** (`fitz`) | Rasterize PDF → ảnh | Nhanh hơn `pdf2image` và **không cần cài poppler ngoài** |
| **OpenCV** | Xử lý ảnh, contour, xấp xỉ đa giác | |
| **NumPy** | Thao tác mảng | |
| **pyclipper** | Offset đa giác theo bán kính dao | Chính xác hơn dilate bằng kernel |
| **Pillow** | Đọc/ghi ảnh phụ trợ | |

---

## Bước 1 — Rasterize PDF

Đầu vào: **3 file PDF tỉ lệ 1:1** đúng như đồ án gốc — mạch in hoàn chỉnh, lỗ khoan, đường viền.
Xuất được từ bất kỳ phần mềm thiết kế nào (Proteus, Altium, KiCad).

**Chọn DPI là quyết định quan trọng nhất của cả pipeline:**

| DPI | 1 pixel | Ảnh bo 180 × 130 mm | Đánh giá |
|---|---|---|---|
| 300 | 84,7 µm | 2126 × 1535 = 3,3 MP | Thô, dễ đứt nét mảnh |
| **600** | **42,3 µm** | **4252 × 3071 = 13,1 MP** | **Chọn** |
| 1200 | 21,2 µm | 8504 × 6142 = 52,2 MP | Thừa, tốn bộ nhớ và thời gian |

Chọn **600 DPI**: 42,3 µm nhỏ hơn nhiều so với khe hở đường mạch điển hình 0,2–0,5 mm, mà vẫn vừa
bộ nhớ và thời gian xử lý.

> **Đây là minh chứng định lượng cho việc nâng lên Raspberry Pi 4.** Ảnh 13 MP × 3 file, cộng các
> bản trung gian (xám, nhị phân, tô đen lỗ khoan, ảnh offset) — **không chạy nổi trên 512 MB của
> Pi Zero 2W**.

**Công thức đổi đơn vị** (giữ nguyên khái niệm PPI của đồ án gốc):

```
1 inch = 25,4 mm
kích thước 1 pixel (mm) = 25,4 / DPI
```

## Bước 2 — Tiền xử lý

```
Ảnh màu → xám → ngưỡng Otsu → nhị phân → morphological opening (khử đốm nhiễu)
```

**Ngưỡng Otsu** thay cho ngưỡng cố định của đồ án gốc: tự thích ứng với độ đậm nhạt khác nhau khi
xuất PDF từ các phần mềm thiết kế khác nhau.

> ## ⚠ RỦI RO SỐ 1 — Đồng bộ 3 file PDF
>
> Ba file PDF **phải cùng gốc tọa độ**. Nếu phần mềm thiết kế xuất lệch nhau dù chỉ **0,2 mm** thì
> **lỗ khoan sẽ trượt khỏi pad** và bo hỏng hoàn toàn — không cứu được.
>
> **Bắt buộc thực hiện:**
> 1. Lấy **bounding box của file đường viền làm chuẩn**
> 2. Crop **cả 3 ảnh theo đúng khung đó**
> 3. **Kiểm tra 3 ảnh có cùng kích thước pixel**, lệch thì báo lỗi và dừng
>
> Không có bước này thì mọi thứ phía sau vô nghĩa. Đồ án gốc không đề cập rủi ro này.

## Bước 3 — Cắt ảnh

PDF xuất trên khổ A4 nên phần lớn diện tích là giấy trắng — phải cắt về vùng bo mạch trước khi xử lý
(đúng như đồ án gốc đã làm). Dùng **chung một khung crop** cho cả 3 ảnh theo cảnh báo ở Bước 2.

## Bước 4 — Trích lỗ khoan

`cv2.findContours` trên layer khoan. Với mỗi contour, lấy bounding box rồi tính tâm theo đúng
công thức của đồ án gốc:

```
Ix = xmin + (xmax − xmin) / 2
Iy = ymin + (ymax − ymin) / 2
d  = max(xmax − xmin,  ymax − ymin)
```

Công thức này đúng cho cả layer tròn lẫn layer chữ nhật nhờ tính đối xứng — nhận xét này là của
đồ án gốc và vẫn giữ nguyên.

**Bổ sung so với đồ án gốc:**

1. **Lượng tử hóa `d` về bộ 6 mũi khoan** có trên máy
2. **Gom nhóm theo mũi khoan** — khoan hết các lỗ cùng cỡ rồi mới thay dao **một lần**

Tối ưu thứ hai cắt trực tiếp số lần thay dao. Mỗi lần ATC tốn vài giây, một bo có thể có 4–5 cỡ lỗ
xen kẽ nhau — không gom nhóm thì máy thay dao hàng chục lần vô ích.

Đầu ra: danh sách `(x, y, tool_id)`.

## Bước 5 — Đường viền

`findContours` trên layer viền, lấy **contour lớn nhất**.

Đồ án gốc đã nhận xét đúng rằng file đường viền thường xuất ra **nhiều hơn một layer**, trong khi
chỉ cần một layer để cắt. Lấy layer có kích thước lớn hơn.

Sau đó `cv2.approxPolyDP` → đa giác kín để cắt.

## Bước 6 — Tô đen lỗ khoan

Giữ nguyên ý tưởng của đồ án gốc: đặt giá trị RGB các pixel thuộc lỗ khoan về **0** trước khi tính
đường phay.

Mục đích: chương trình sẽ **không phay** ở chỗ sắp khoan. Không có bước này, máy tốn thêm thời gian
phay đường quanh từng layer lỗ khoan một cách vô ích.

## Bước 7 — Đường phay cách ly ⭐ *khâu thay đổi hoàn toàn*

```
1. findContours (RETR_CCOMP)     → biên dạng vùng đồng kèm lỗ bên trong
2. pyclipper offset              → dịch biên dạng ra ngoài bằng BÁN KÍNH DAO
3. cv2.approxPolyDP              → rút gọn Douglas–Peucker
```

**Hiệu quả rút gọn — đây là toàn bộ lý do chuyển được sang PLC:**

| | Đồ án gốc | Bản polyline |
|---|---|---|
| 1 biên dạng ~5000 px | **5000 lệnh `bit`** | **~200–400 lệnh `line`** |
| Truyền qua Ethernet | Không khả thi | Vài giây |

> ## ⚠ RỦI RO SỐ 2 — Chặn trên của epsilon
>
> Douglas–Peucker đảm bảo sai lệch tối đa **≤ epsilon** so với đường gốc. Do đó bắt buộc:
>
> **epsilon < một nửa khe hở nhỏ nhất trên bo**
>
> Vượt ngưỡng này, đường phay sau khi đơn giản hóa sẽ **ăn vào đường mạch bên cạnh**.
> Khuyến nghị epsilon ≈ 1 px (42 µm ở 600 DPI) và kiểm tra lại bằng Preview.

> ## ⚠ Kiểm tra khả thi trước khi chạy
>
> Nếu **đường kính dao lớn hơn khe hở giữa hai đường mạch** thì không thể phay cách ly được.
> Phần mềm phải **phát hiện và cảnh báo trước khi máy chạy**, chứ không để máy phay hỏng bo rồi
> mới biết.

## Bước 8 — Phay mặt

**Zig-zag**: cắt các đường ngang theo vùng cần bóc đồng. Mỗi đường chỉ là **1 đoạn thẳng**, rất hợp
với mô hình lệnh của PLC.

Đơn giản và nhanh hơn contour-parallel (offset vào trong nhiều lớp), vốn sinh ra nhiều đoạn cong phức tạp.

## Bước 9 — Đổi tọa độ

```
x_mm = x_px × 25,4 / DPI
y_mm = (H_px − y_px) × 25,4 / DPI       ← LẬT TRỤC Y
```

Sau đó cộng offset vùng gia công người dùng chọn ở tab **Select Area**, rồi xuất ra **µm kiểu `DInt`**
cho Data Block của PLC.

> ## ⚠ RỦI RO SỐ 3 — Lật trục Y
>
> Ảnh có gốc tọa độ ở **trên-trái**, máy có gốc ở **dưới-trái**. Quên lật trục Y sẽ cho ra
> **bo mạch bị lật gương** — lỗi kinh điển của loại phần mềm này, và chỉ phát hiện được sau khi
> đã phay xong.

## Bước 10 — Sắp thứ tự đường chạy

Nearest-neighbour trên các điểm đầu polyline để giảm quãng di chuyển không cắt.

Đồ án gốc không có bước này. Bổ sung được thì giảm đáng kể thời gian gia công, vì máy chạy
20 mm/s khi di chuyển và đường chạy tùy tiện có thể tốn nhiều hơn cả thời gian phay.

## Bước 11 — Chia nhỏ đoạn để bù cao độ

Sau khi có leveling map 66 điểm (ma trận 6×11), **mọi đoạn dài hơn ~5 mm phải được chia nhỏ** để z
bám theo bề mặt cong của phíp đồng.

> ## ⚠ RỦI RO SỐ 4 — Không chia nhỏ đoạn
>
> Một đoạn thẳng dài chỉ có z ở hai đầu sẽ **phay hụt ở giữa hoặc cắt đứt mạch** — đúng vấn đề mà
> leveling map sinh ra để giải quyết. Bỏ qua bước này thì 66 điểm đo trở nên vô nghĩa.

Bước này chạy **trên Pi lúc biên dịch**, không phải trong PLC — xem `chuc-nang-plc.md` §1.

## Bước 12 — Xuất Acode

```
; header
; DPI=600  BOARD=180x130  TOOLS=6  SEGMENTS=1842

rapid     x,y          ; di chuyển nhanh, dao nâng
zdown                  ; hạ dao
line      x,y,f        ; THAY cho "bit x,y" của bản gốc
zup                    ; nâng dao
movedrill x,y,tool
movecut   x,y
change    tool
```

So với tập lệnh gốc `move` / `bit` / `movecut` / `movedrill` / `change`: giữ nguyên 3 lệnh cuối,
thay `bit` bằng `line`, tách `move` thành `rapid` + `zdown`/`zup` cho rõ ràng.

---

## Kiểm chứng

**Vẽ ngược polyline đã sinh lên ảnh gốc và chồng lên nhau** để soi sai lệch. Tận dụng luôn tab
**Preview** đã có sẵn trong phần mềm gốc — nay kiêm thêm vai trò kiểm tra độ trung thực của phép
rút gọn Douglas–Peucker.

**Ước lượng thời gian xử lý trên Pi 4:** ảnh 13 MP, contour + offset + rút gọn ≈ **10–60 s cho một
bo**. Đây là **ước lượng, cần đo thực tế**.

---

## Bảng rủi ro — xếp theo mức nghiêm trọng

| # | Rủi ro | Hậu quả | Phòng ngừa |
|---|---|---|---|
| 1 | **Lệch 3 file PDF** | Lỗ khoan trượt khỏi pad, **hỏng bo hoàn toàn** | Crop chung theo bounding box file đường viền, kiểm tra cùng kích thước pixel |
| 2 | **Epsilon quá lớn** | Đường phay ăn vào mạch bên cạnh | epsilon < nửa khe hở nhỏ nhất (~1 px) |
| 3 | **Quên lật trục Y** | Bo mạch lật gương | Kiểm tra công thức `y = (H_px − y_px) × 25,4/DPI` |
| 4 | **Không chia nhỏ đoạn** | Leveling map mất tác dụng, phay hụt hoặc đứt mạch | Chia mọi đoạn > 5 mm |

Ba trong bốn rủi ro này không được đề cập trong đồ án gốc.

---

## Tài liệu liên quan

| File | Nội dung |
|---|---|
| [`thiet-bi-va-chuc-nang.md`](thiet-bi-va-chuc-nang.md) | Danh sách thiết bị và chức năng tổng thể |
| [`chuc-nang-plc.md`](chuc-nang-plc.md) | Đặc tả PLC: I/O, khối hàm, DB, an toàn, đấu dây |
| [`so-sanh-stm32-vs-s7.md`](so-sanh-stm32-vs-s7.md) | Đối chiếu bản gốc ↔ bản PLC |
