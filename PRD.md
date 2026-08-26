# PRD — Hệ thống gia công PCB tự động tích hợp AOI

## 1. Tổng quan

Hệ thống được thiết kế để tự động hóa quá trình gia công PCB từ dữ liệu thiết kế đến kiểm tra chất lượng.

Luồng chính:

```text
Gerber
  ↓
NC Processing
  ↓
Raspberry Pi 4
  ↓
S7-1200 PLC
  ↓
Motor & Motion System
  ↓
PCB Processing
  ↓
Camera / AOI Inspection
```

Raspberry Pi 4 đảm nhiệm xử lý cấp cao, giao tiếp, xử lý dữ liệu và AOI. S7-1200 PLC đảm nhiệm điều khiển thời gian thực, I/O và chuyển động.

---

## 2. Mục tiêu sản phẩm

| ID | Mục tiêu |
|---|---|
| G1 | Tự động hóa quá trình gia công PCB |
| G2 | Chuyển dữ liệu Gerber thành dữ liệu điều khiển gia công |
| G3 | Điều khiển chính xác các cơ cấu chuyển động |
| G4 | Tích hợp AOI để kiểm tra chất lượng PCB |
| G5 | Thay thế kiến trúc STM32 bằng hệ thống PLC công nghiệp |

---

## 3. Người dùng mục tiêu

Hệ thống hướng đến:

- Kỹ sư điện tử và tự động hóa.
- Phòng thí nghiệm và môi trường đào tạo.
- Nhóm phát triển và thử nghiệm PCB.
- Người vận hành cần gia công và kiểm tra PCB bán tự động hoặc tự động.

---

## 4. Yêu cầu chức năng

### FR-01 — Nhập dữ liệu

Hệ thống phải hỗ trợ tiếp nhận dữ liệu Gerber và các dữ liệu liên quan đến thiết kế PCB.

### FR-02 — Xử lý dữ liệu

Hệ thống phải chuyển đổi dữ liệu thiết kế thành dữ liệu NC hoặc dữ liệu phù hợp cho quá trình điều khiển gia công.

### FR-03 — Điều khiển gia công

Hệ thống phải hỗ trợ 5 quy trình gia công theo thiết kế kỹ thuật của máy.

Mỗi quy trình phải:

- Có điều kiện khởi động rõ ràng.
- Có trạng thái thực hiện.
- Có điều kiện hoàn thành.
- Có cơ chế dừng và xử lý lỗi.

### FR-04 — Điều khiển chuyển động

PLC phải điều khiển các cơ cấu chuyển động và các tín hiệu I/O liên quan đến quá trình gia công.

Các chức năng chuyển động phải tuân thủ giới hạn phần cứng của S7-1200.

### FR-05 — AOI

Hệ thống phải hỗ trợ kiểm tra quang học tự động sau hoặc trong quá trình gia công.

AOI phải đánh giá 5 chỉ tiêu chất lượng:

- R1
- R2
- R3
- R4
- R5

Định nghĩa, thuật toán và ngưỡng đánh giá chi tiết được xác định trong tài liệu AOI chuyên biệt.

### FR-06 — Hiệu chuẩn

Hệ thống phải hỗ trợ quy trình hiệu chuẩn để đảm bảo sự tương quan giữa:

- Tọa độ phần mềm.
- Hệ tọa độ máy.
- Vị trí cơ cấu chuyển động.
- Hệ thống camera.

### FR-07 — Phần mềm PC

Phần mềm vận hành phải cung cấp 7 nhóm chức năng hoặc tab chính theo thiết kế giao diện của hệ thống.

Các tab phải hỗ trợ các hoạt động như:

- Quản lý dữ liệu.
- Thiết lập gia công.
- Điều khiển máy.
- Giám sát trạng thái.
- Hiệu chuẩn.
- AOI.
- Cấu hình và chẩn đoán.

---

## 5. Yêu cầu phi chức năng

### NFR-01 — An toàn

Hệ thống phải hỗ trợ cơ chế dừng an toàn và ngăn thực hiện chuyển động không hợp lệ.

### NFR-02 — Độ tin cậy

Các trạng thái hệ thống phải được kiểm soát bằng state machine và không được tạo transition không được định nghĩa.

### NFR-03 — Chống nhiễu

Thiết kế điện và tín hiệu phải xem xét ảnh hưởng của nhiễu từ động cơ, nguồn và thiết bị công nghiệp.

### NFR-04 — Nhiệt độ

Các thành phần điện tử và nguồn phải hoạt động trong giới hạn nhiệt cho phép.

### NFR-05 — Giao tiếp

Raspberry Pi 4 và S7-1200 phải duy trì cơ chế giao tiếp ổn định giữa tầng xử lý cấp cao và tầng điều khiển thời gian thực.

---

## 6. Kiến trúc hệ thống

### Raspberry Pi 4

Chịu trách nhiệm:

- Xử lý dữ liệu Gerber/NC.
- Giao tiếp cấp cao.
- Điều phối quy trình.
- Xử lý AOI.
- Xử lý ảnh.
- Tính toán nặng.
- Giao diện và quản lý dữ liệu.

### S7-1200 PLC

Chịu trách nhiệm:

- Điều khiển thời gian thực.
- I/O.
- Điều khiển chuyển động.
- PTO.
- PWM.
- Thực thi trình tự máy.
- Giám sát tín hiệu phần cứng.

### Nguyên tắc phân chia

Không chuyển các tác vụ yêu cầu tính xác định thời gian thực sang Raspberry Pi nếu không có lý do kỹ thuật rõ ràng.

Không chuyển các tác vụ xử lý ảnh hoặc tính toán nặng sang PLC.

---

## 7. Yêu cầu phần cứng bắt buộc

Các thành phần và điều kiện sau phải được kiểm tra theo tài liệu thiết kế:

- Điện trở 2 kΩ theo vị trí và chức năng đã xác định.
- Phiên bản nguồn DC/DC/DC phải đúng với thiết kế được phê duyệt.
- Camera phải sử dụng cấu hình ống kính M12 theo yêu cầu hệ thống.
- Các tín hiệu chuyển động phải tuân thủ giới hạn PTO của PLC.
- Timing PWM phải được kiểm chứng trong điều kiện vận hành thực tế 24 V.
- Tín hiệu probe tốc độ cao phải sử dụng cơ chế hardware interrupt phù hợp.

Không được thay đổi các yêu cầu này mà không thực hiện đánh giá kỹ thuật.

---

## 8. AOI và quản lý bộ nhớ

Hệ thống AOI xử lý ảnh theo cơ chế chia thành 16 tile.

Quy trình:

```text
Load Tile
   ↓
Process
   ↓
Extract Result
   ↓
Release Memory
   ↓
Next Tile
```

Không được giữ toàn bộ 16 tile trong RAM nếu chưa có đánh giá xác nhận khả năng bộ nhớ.

---

## 9. Kiểm chứng bắt buộc trước vận hành

Trước khi hệ thống được đưa vào vận hành, phải hoàn thành 4 điểm kiểm chứng bắt buộc:

1. Kiểm chứng giới hạn PTO và cấu hình tín hiệu chuyển động.
2. Đo và xác nhận timing PWM trong điều kiện nguồn 24 V thực tế.
3. Kiểm chứng tín hiệu probe bằng cơ chế ngắt phần cứng.
4. Kiểm chứng cấu hình AOI, camera, quang học và khả năng quản lý bộ nhớ.

Bất kỳ điểm nào chưa được xác nhận phải được đánh dấu rõ ràng và không được coi là đã hoàn thành.

---

## 10. State Machine

Hệ thống hoạt động theo state machine gồm 8 trạng thái, từ:

`IDLE → ... → INSPECTING`

Mọi chức năng mới phải xác định:

- State áp dụng.
- Điều kiện bắt đầu.
- Điều kiện kết thúc.
- Transition hợp lệ.
- Hành vi khi xảy ra lỗi.
- Cơ chế reset hoặc recovery.

Không được tạo transition ngầm hoặc thay đổi state machine mà không cập nhật tài liệu.

---

## 11. So sánh kiến trúc STM32 và PLC

| Tiêu chí | Kiến trúc STM32 | Kiến trúc PLC mới |
|---|---|---|
| Điều khiển chính | Firmware MCU | S7-1200 PLC |
| Phát triển | Embedded programming | PLC engineering |
| Real-time | MCU firmware | PLC deterministic control |
| I/O | MCU peripherals | Industrial I/O |
| Motion | Firmware implementation | PTO / PLC control |
| Mở rộng | Phụ thuộc hardware design | Dễ tích hợp công nghiệp |
| Chẩn đoán | Debug firmware | PLC diagnostics |
| Bảo trì | Embedded expertise | Industrial automation workflow |

---

## 12. Tiêu chí hoàn thành

Sản phẩm được xem là đạt yêu cầu khi:

- [ ] Dữ liệu Gerber được xử lý thành dữ liệu phục vụ gia công.
- [ ] 5 quy trình gia công hoạt động theo trình tự được xác định.
- [ ] Raspberry Pi 4 và S7-1200 giao tiếp ổn định.
- [ ] Các cơ cấu chuyển động hoạt động trong giới hạn phần cứng.
- [ ] AOI kiểm tra đủ 5 chỉ tiêu R1–R5.
- [ ] Quy trình hiệu chuẩn được thực hiện thành công.
- [ ] Phần mềm PC cung cấp đầy đủ 7 nhóm chức năng.
- [ ] Hoàn thành 4 điểm kiểm chứng bắt buộc.
- [ ] State machine hoạt động với các transition được xác định.
- [ ] Các yêu cầu an toàn, nhiễu, nhiệt và giao tiếp được đánh giá trước vận hành.

---

## Nguyên tắc

**PRD xác định hệ thống cần đạt được điều gì.**

Các thông số kỹ thuật chi tiết, sơ đồ phần cứng, thuật toán AOI, state transition cụ thể và quy trình triển khai được lưu trong các tài liệu kỹ thuật chuyên biệt.
