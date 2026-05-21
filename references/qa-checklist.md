# QA Checklist — Đối chiếu bản dịch với bản gốc

## Quy trình QA

### Bước 1: Fetch cả hai bản

**Bản gốc** — mở lại URL WeChat qua CDP, chạy Script 2 (text + markers) + Script 4 (videos):
```bash
curl -s -X POST --data-raw "WECHAT_URL" "http://localhost:3456/new"
# scroll + eval Script 2, Script 4 từ wechat-extraction.md
```

**Bản dịch** — fetch từ Lark (v2 API):
```bash
lark-cli docs +fetch --api-version v2 --doc DOC_ID --doc-format xml
```

### Bước 2: Đối chiếu tự động bằng Python

```python
import re, json, sys
data = json.load(sys.stdin)
content = data['data']['document']['content']

stats = {
  'images':  len(re.findall(r'<img[^>]+>', content)),
  'videos':  len(re.findall(r'<figure[^>]*view-type="Preview"', content)),
  'h2':      len(re.findall(r'<h2[^>]*>', content)),
  'h3':      len(re.findall(r'<h3[^>]*>', content)),
  'markers': len(re.findall(r'\[\[IMG_\d+\]\]', content)),
}
print(stats)
```

| Hạng mục | Cách kiểm | Ngưỡng chấp nhận |
|----------|----------|------------------|
| Số `<img>` | đếm trong XML | = số ảnh từ Phase 1.3 |
| Số `<figure view-type="Preview">` | đếm trong XML | = số video từ Phase 1.4 |
| Số heading `##` | gốc vs dịch | bằng nhau |
| Số heading `###` | gốc vs dịch | bằng nhau (±2 do tổ chức lại) |
| Số đoạn văn | gốc vs dịch | chênh lệch < 15% |
| Marker còn sót | `[[IMG_` trong Lark doc | = 0 |

### Bước 3: Kiểm tra vị trí ảnh & video

**Với mỗi `<img>` trong Lark doc:**
1. Lấy 50 ký tự trước và sau trong bản dịch
2. So với `textBefore` / `textAfter` trong image-context map (Phase 2)
3. Ngữ cảnh phải tương ứng

**Với mỗi `<figure view-type="Preview">`:**
1. Tìm paragraph ngay trước → kiểm tra có phải caption (`📺 ...`) hoặc anchor text khớp Phase 2 không
2. Nếu video chèn ở vị trí sai → dùng `block_move_after` hoặc xoá-chèn lại

### Bước 4: Kiểm tra nội dung thiếu

Đọc bản gốc theo từng section heading, đối chiếu:
- Mọi ý chính đều có mặt
- Ví dụ / case study không bị bỏ sót
- Trích dẫn nguyên văn (nếu có) được giữ lại
- Tên Skill / sản phẩm giữ nguyên tiếng Anh

### Bước 5: Kiểm tra format Lark

- [ ] Callout đóng mở đúng
- [ ] Grid đóng mở đúng, số column khớp
- [ ] Table render được (không bị lỗi cú pháp)
- [ ] Ảnh hiển thị được (đã upload vào CDN Lark, URL `internal-api-drive-stream-sg.larksuite.com`)
- [ ] Video render preview player (xác nhận `view-type="Preview"` trong XML)
- [ ] Không có raw HTML tag hiện ra trên trang
- [ ] Không có `[[IMG_N]]` lộ ra trong text

---

## Bảng báo cáo QA mẫu

```markdown
## Báo cáo QA — [TÊN BÀI]

| Hạng mục | Gốc | Dịch | Trạng thái |
|----------|-----|------|-----------|
| Ảnh (<img>) | 9 | 9 | ✅ |
| Video (<figure Preview>) | 8 | 8 | ✅ |
| Heading H2 | 6 | 6 | ✅ |
| Heading H3 | 8 | 10 | ⚠️ (+2 do tách section Bonus) |
| Marker [[IMG_]] còn sót | - | 0 | ✅ |
| Đoạn thiếu nội dung | - | 0 | ✅ |

🔗 Tài liệu: https://YOUR_TENANT.sg.larksuite.com/docx/DOC_ID
```

---

## Vấn đề thường gặp và cách sửa

| Vấn đề | Nguyên nhân | Cách sửa |
|--------|-------------|----------|
| Ảnh hoàn toàn không xuất hiện | Dùng `<image>` markdown ở Phase 4 (bị strip v2) | Chèn lại qua Phase 4a (`block_insert_after` + XML `<img>`) |
| Thiếu vài ảnh | Chưa scroll bottom trước khi extract | Mở lại bài, scroll, extract lại; dùng `block_insert_after` chèn ảnh thiếu |
| Ảnh sai thứ tự | Insert nhiều ảnh sau cùng anchor → đảo ngược | Insert lại với anchor khác nhau hoặc theo thứ tự ngược |
| Video không render player | Quên `--file-view preview` | Xoá block, insert lại với `--file-view preview` |
| Video URL 403 | Auth token expire | Mở bài WeChat ngay, lấy URL mới, tải liền |
| `block_insert_after` trên file block fail | Schema không cho insert sau file block | Insert caption TRƯỚC khi insert video |
| Thiếu đoạn văn | Dịch bỏ sót section | `docs +update --command block_insert_after` chèn đoạn thiếu |
| `[[IMG_N]]` còn lộ trong text | Quên xoá markers ở Phase 4 | `str_replace --pattern "\\[\\[IMG_\\d+\\]\\]" --content ""` |
| Callout bị vỡ | Thiếu tag đóng | `str_replace` sửa hoặc xoá + chèn lại block |

---

## Khi nào QA đạt?

- ✅ Tất cả heading khớp
- ✅ Tất cả ảnh nội dung chính đã chèn và đúng vị trí
- ✅ Tất cả video đã chèn với `view-type="Preview"` (inline player)
- ✅ Không thiếu đoạn nội dung quan trọng
- ✅ Không còn `[[IMG_N]]` marker trong doc
- ✅ Format Lark render đúng (callout, grid, table, image, video)
- ✅ Tên Skill/sản phẩm giữ nguyên tiếng Anh
