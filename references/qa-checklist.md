# QA Checklist — Đối chiếu bản dịch với bản gốc

## Quy trình QA

### Bước 1: Fetch cả hai bản

**Bản gốc** — mở lại URL WeChat qua CDP, chạy Script 2 (text + markers):
```bash
curl -s "http://localhost:3456/new?url=WECHAT_URL"
# scroll + eval Script 2 từ wechat-extraction.md
```

**Bản dịch** — fetch từ Lark:
```bash
lark-cli docs +fetch --doc DOC_ID
```

### Bước 2: Đối chiếu tự động

| Hạng mục | Cách kiểm | Ngưỡng chấp nhận |
|----------|----------|------------------|
| Số heading `##` | Đếm trong gốc vs dịch | Bằng nhau |
| Số heading `###` | Đếm trong gốc vs dịch | Bằng nhau |
| Số ảnh | Đếm `[[IMG_N]]` gốc vs `<image` dịch | Bằng nhau (hoặc giải thích ảnh bỏ qua) |
| Số đoạn văn | Đếm paragraph gốc vs dịch | Chênh lệch < 15% |
| Marker còn sót | Tìm `[[IMG_` trong Lark doc | Phải = 0 |

### Bước 3: Kiểm tra vị trí ảnh

Với mỗi `<image>` trong Lark doc:
1. Lấy 50 ký tự trước và sau ảnh trong bản dịch
2. So với `textBefore` / `textAfter` trong image-context map (Phase 2)
3. Ngữ cảnh phải tương ứng (nội dung dịch khớp với nội dung gốc xung quanh ảnh)

### Bước 4: Kiểm tra nội dung thiếu

Mở bản gốc theo từng section heading:

1. Đọc section gốc
2. Tìm section tương ứng trong bản dịch
3. Kiểm tra:
   - Mọi ý chính đều có mặt
   - Ví dụ / case study không bị bỏ sót
   - Trích dẫn nguyên văn (nếu có) được giữ lại
   - Tên Skill / sản phẩm giữ nguyên

### Bước 5: Kiểm tra format Lark

- [ ] Callout đóng mở đúng
- [ ] Grid đóng mở đúng, số column khớp
- [ ] Table render được (không bị lỗi cú pháp)
- [ ] Ảnh hiển thị được (URL không lỗi `&amp;`)
- [ ] Không có raw HTML tag hiện ra trên trang

---

## Bảng báo cáo QA mẫu

```markdown
## Báo cáo QA — [TÊN BÀI]

| Hạng mục | Gốc | Dịch | Trạng thái |
|----------|-----|------|-----------|
| Heading ## | 5 | 5 | ✅ |
| Heading ### | 8 | 8 | ✅ |
| Ảnh | 18 | 16 | ⚠️ Thiếu 2 (IMG_9, IMG_16 — ảnh phụ) |
| Đoạn văn | 42 | 44 | ✅ (+2 do tách đoạn dài) |
| Marker còn sót | - | 0 | ✅ |
| Đoạn thiếu nội dung | - | 1 | ⚠️ Thiếu đoạn @张咋啦 |

### Chi tiết vấn đề:
1. **Thiếu đoạn @张咋啦**: Bản gốc có đoạn về PPT美化 Skill → cần bổ sung
2. **IMG_9, IMG_16**: Ảnh minh hoạ phụ, không ảnh hưởng nội dung → chấp nhận

### Hành động sửa:
- `docs +update --mode insert_before` để chèn đoạn thiếu
- `docs +update --mode insert_after` để chèn ảnh thiếu
```

---

## Vấn đề thường gặp và cách sửa

| Vấn đề | Nguyên nhân | Cách sửa |
|--------|-------------|----------|
| Thiếu đoạn văn | Dịch bỏ sót section phụ | `docs +update --mode insert_before/after` chèn đoạn thiếu |
| Thiếu ảnh | Không chèn đủ `<image>` | `docs +update --mode insert_before/after` chèn ảnh |
| Ảnh sai vị trí | Chèn không khớp context | `docs +update --mode replace_range` di chuyển ảnh |
| Ảnh không hiển thị | URL chứa `&amp;` | Sửa URL: replace `&amp;` → `&` |
| Callout bị vỡ | Thiếu tag đóng `</callout>` | `docs +update --mode replace_range` sửa tag |
| Nội dung bị cắt | Chunk quá lớn khi tạo | Xoá doc, tạo lại với chunk nhỏ hơn |
| `[[IMG_N]]` còn sót | Quên thay marker | `docs +update --mode replace_all` thay bằng `<image>` |

---

## Khi nào QA đạt?

- ✅ Tất cả heading khớp
- ✅ Tất cả ảnh nội dung chính đã chèn và đúng vị trí  
- ✅ Không thiếu đoạn nội dung quan trọng
- ✅ Không còn `[[IMG_N]]` marker trong doc
- ✅ Format Lark render đúng (callout, grid, table, image)
- ✅ Tên Skill/sản phẩm giữ nguyên tiếng Anh
