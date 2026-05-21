# Template và kỹ thuật format Lark document (v2 API)

⚠️ **lark-cli v2 API** — luôn dùng `--api-version v2`. Tham số khác v1:
- v2: `--content "@file" --doc-format markdown|xml`
- v1: `--markdown "@file"` (deprecated)

## Template metadata header

Mỗi bài dịch bắt đầu bằng metadata callout hoặc blockquote ở đầu bài (sau title):

```markdown
> **Nguồn:** Bài viết gốc tiếng Trung trên WeChat — [TÊN_BÀI](URL_GỐC) | Tác giả: TÊN | Dịch sang tiếng Việt bởi AI
```

**Lưu ý:** Không lặp lại title bài trong markdown — `--title` đã là tiêu đề document.

---

## ⚠️ Chèn ảnh: KHÔNG dùng markdown — phải XML 2-pass

### Tại sao markdown image không hoạt động ở v2

Cú pháp `<image url="..."/>` (lark-flavored markdown) **bị strip silently** khi tạo doc qua `--doc-format markdown` ở v2 API. Document được tạo thành công nhưng KHÔNG có ảnh nào.

### Pattern đúng: 2-pass

**Pass 1 (Phase 4):** Tạo doc text-only — bỏ hết `[[IMG_N]]` markers, không nhúng image tags.

**Pass 2 (Phase 4a):** Sau khi doc tồn tại, fetch block IDs và insert ảnh qua `block_insert_after` với XML.

### Chèn ảnh qua block_insert_after (v2)

```bash
echo '<img src="IMAGE_URL" align="center"/>' | \
lark-cli docs +update \
  --api-version v2 \
  --doc DOC_ID \
  --command block_insert_after \
  --block-id ANCHOR_BLOCK_ID \
  --doc-format xml \
  --content -
```

**Tag:** `<img>` (XML), không phải `<image>`.

**Attribute:**
- `src="..."` — URL ảnh trực tiếp (WeChat `mmbiz.qpic.cn` công khai, Lark tự download và upload vào CDN nội bộ)
- `align="center"` — căn giữa (mặc định)

### Quy tắc thứ tự khi insert nhiều ảnh

`block_insert_after` chèn block mới ngay sau anchor — nếu chèn 2 ảnh sau cùng một anchor:
```
[Anchor] → [Ảnh 2 (chèn sau)] → [Ảnh 1 (chèn trước)] → [Next block]
```
Thứ tự bị **đảo ngược**. Cách xử lý:

1. **Dùng anchor khác nhau** (paragraph khác nhau làm anchor cho mỗi ảnh)
2. **Insert theo thứ tự ngược** (cuối doc trước, đầu doc sau)
3. **Chain anchor:** lấy block_id của ảnh vừa insert làm anchor cho ảnh tiếp theo

### Ảnh nào nên chèn, ảnh nào bỏ?

- **Chèn**: Banner, ảnh minh hoạ nội dung, screenshot, biểu đồ, infographic
- **Có thể bỏ**: Avatar tác giả, icon trang trí nhỏ, QR code cuối bài (chỉ giữ nếu là CTA quan trọng)

---

## Callout patterns

| Mục đích | Emoji | Background |
|----------|-------|------------|
| Insight quan trọng / tip | 💡 | light-blue |
| Cảnh báo / lưu ý | ⚠️ | light-yellow |
| Lỗi / nguy hiểm | ❌ | light-red |
| Kết luận / thành công | ✅ | light-green |
| Trích dẫn gốc / quote | 📌 | light-purple |
| Tóm tắt / overview | 📖 | pale-gray |

```html
<callout emoji="💡" background-color="light-blue">
Nội dung insight quan trọng ở đây. Hỗ trợ **bold** và *italic*.
</callout>
```

**Hạn chế của callout:** Chỉ chứa text, heading, list, quote. Không chứa code block, table, image.

---

## Grid (phân cột)

Dùng khi có nội dung so sánh song song hoặc cần bố cục 2-3 cột:

```html
<grid cols="2">
<column>

Nội dung cột trái

</column>
<column>

Nội dung cột phải

</column>
</grid>
```

**Lưu ý:** Phải có dòng trống trước và sau nội dung trong `<column>`.

---

## Bảng

### Bảng đơn giản (Markdown)

```markdown
| Cột 1 | Cột 2 | Cột 3 |
|-------|-------|-------|
| Data  | Data  | Data  |
```

### Bảng phức tạp (lark-table)

Dùng khi cell chứa list, code block, hoặc nội dung nhiều dòng — xem skill `lark-doc` để biết chi tiết.

---

## Lệnh tạo và update document

### Tạo (Phase 4)

```bash
# Bắt buộc: file phải nằm trong thư mục hiện tại với relative path
cat > ./article_vi.md << 'MDEOF'
[FULL MARKDOWN CONTENT — text only, không có <image>]
MDEOF

lark-cli docs +create \
  --api-version v2 \
  --title "TIÊU_ĐỀ" \
  --doc-format markdown \
  --content "@./article_vi.md"
```

**Tham số tuỳ chọn:**
- `--parent-token TOKEN` — đặt vào folder/wiki cụ thể
- `--parent-position my_library` — đặt vào personal library

### Append section bổ sung

```bash
lark-cli docs +update \
  --api-version v2 \
  --doc DOC_ID \
  --command append \
  --doc-format markdown \
  --content "@./chunk_2.md"
```

### Sửa text (str_replace)

```bash
lark-cli docs +update \
  --api-version v2 \
  --doc DOC_ID \
  --command str_replace \
  --pattern "regex pattern" \
  --content "replacement text"
```

### Xoá block

```bash
lark-cli docs +update \
  --api-version v2 \
  --doc DOC_ID \
  --command block_delete \
  --block-id BLOCK_ID
```

---

## Chiến lược chia chunk cho bài dài

Bài v2 API không có giới hạn rõ ràng về độ dài content. Nhưng để dễ debug và update, có thể:

1. Tạo chunk 1 với `+create`
2. Append các chunk còn lại bằng `+update --command append`

Quy tắc:
1. Bẻ tại heading boundary (`##` hoặc `###`) — không bẻ giữa đoạn
2. Không bẻ giữa callout/grid đang mở — mỗi chunk phải có HTML tag đóng đầy đủ
3. Chunk đầu tiên: metadata header + nội dung đến hết section đầu tiên

---

## Xử lý ký tự đặc biệt trong shell

- **Quote/heredoc hỗn loạn:** luôn lưu content ra file rồi dùng `@./file.md`. Tránh nhúng markdown lớn vào shell command.
- **Tên file có ký tự đặc biệt (tiếng Việt, emoji):** vẫn dùng relative path đơn giản như `./article.md`, không cần escape.

---

## Checklist format trước khi tạo

- [ ] Title không lặp lại trong markdown body
- [ ] Metadata header (blockquote/callout) ở đầu bài
- [ ] `---` phân cách giữa các section lớn
- [ ] Đã **xoá** tất cả `[[IMG_N]]` markers (sẽ chèn lại ở Phase 4a qua XML)
- [ ] Caption video (`**📺 ...:**`) giữ lại làm anchor cho Phase 4b
- [ ] Callout, grid, table đều đóng tag đúng
- [ ] Không có dòng trống thừa liên tiếp (> 2 dòng)
- [ ] File markdown lưu ở thư mục hiện tại với tên relative (`./article_vi.md`)
