# Hình ảnh lưu đồ và sơ đồ

Xuất từ mã nguồn Mermaid trong các tài liệu, bằng `@mermaid-js/mermaid-cli`.

**Mỗi lưu đồ có hai định dạng:**

| Định dạng | Dùng khi nào |
|---|---|
| `.png` | Chèn vào Word, PowerPoint. Xuất ở **scale 3×** nên nét vẫn sắc khi phóng to |
| `.svg` | **Nên dùng khi in báo cáo** — vector, không vỡ nét ở mọi cỡ giấy |

> **Không sửa trực tiếp file ảnh.** Nguồn là các khối ```mermaid``` trong tài liệu `.md`.
> Sửa nguồn rồi xuất lại, nếu không hình và tài liệu sẽ lệch nhau.

## Danh sách

| Tên file | Thuộc tài liệu | Cỡ PNG |
|---|---|---|
| `aoi-01-4-3-luu-do-tong-the` | Kiểm tra quang học | 280 KB |
| `gerber-01-3-luu-do-tong-the` | Gerber → NC | 203 KB |
| `luudo-01-1-van-hanh-tong-the` | Lưu đồ giải thuật | 291 KB |
| `luudo-02-2-bien-dich-gerber-chuong-trinh-chay-may` | Lưu đồ giải thuật | 272 KB |
| `luudo-03-3-set-home` | Lưu đồ giải thuật | 180 KB |
| `luudo-04-4-leveling-map-66-diem` | Lưu đồ giải thuật | 249 KB |
| `luudo-05-5-noi-suy-duong-thang-2-truc` | Lưu đồ giải thuật | 219 KB |
| `luudo-06-6-thay-dao-tu-dong` | Lưu đồ giải thuật | 214 KB |
| `luudo-07-7-bat-tay-truyen-du-lieu` | Lưu đồ giải thuật | 234 KB |
| `luudo-08-8-kiem-tra-quang-hoc-sau-gia-cong` | Lưu đồ giải thuật | 263 KB |
| `luudo-09-9-dinh-vi-phoi-va-toi-uu-vi-tri-dat` | Lưu đồ giải thuật | 303 KB |
| `luudo-10-10-may-trang-thai-he-thong` | Lưu đồ giải thuật | 219 KB |
| `phoi-01-10-luu-do` | Định vị & tối ưu phôi | 335 KB |
| `plc-01-8-1-so-do-khoi-toan-he-thong` | Đặc tả PLC | 122 KB |

## Lệnh xuất lại

```bash
npm install @mermaid-js/mermaid-cli
echo '{"args":["--no-sandbox"],"executablePath":"<đường-dẫn-chromium>"}' > pptr.json

# PNG cho Word / PowerPoint
mmdc -i so-do.mmd -o so-do.png -p pptr.json -b white -s 3

# SVG cho bản in
mmdc -i so-do.mmd -o so-do.svg -p pptr.json -b white
```

## Lưu ý cú pháp đã gặp

Hai lỗi thật phát hiện được nhờ khâu xuất ảnh — chúng cũng làm **hỏng hiển thị trên GitHub**:

| Lỗi | Nguyên nhân | Cách sửa |
|---|---|---|
| `Parse error ... got 'DEFAULT'` | `linkStyle` **không hợp lệ** trong `stateDiagram-v2`, chỉ dùng được với `flowchart` | Bỏ `linkStyle`, dùng `classDef` + `class` |
| `Parse error ... got 'STYLE_SEPARATOR'` | Dòng `subgraph` **không nhận** `:::class` | Bỏ `:::class` khỏi dòng `subgraph`, tô màu bằng `style <tên> fill:...` |
