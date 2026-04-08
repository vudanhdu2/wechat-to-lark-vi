# Template và kỹ thuật format Lark document

## Template metadata header

Mỗi bài dịch bắt đầu bằng callout giới thiệu nguồn:

```html
<callout emoji="📖" background-color="light-blue">
*Dịch từ bài viết gốc tiếng Trung trên WeChat — Chủ đề: [CHỦ_ĐỀ]*

Link bài gốc: [TÊN_BÀI](URL_GỐC)
</callout>
```

**Lưu ý:** Không lặp lại title bài trong markdown — `--title` đã là tiêu đề document.

---

## Chèn ảnh

### Cú pháp

```html
<image url="https://mmbiz.qpic.cn/..." align="center"/>
```

### Quy tắc

- Luôn dùng `align="center"` cho ảnh bài viết
- URL ảnh WeChat (`mmbiz.qpic.cn`) công khai, dùng trực tiếp — Lark tự download
- Thay `[[IMG_N]]` bằng `<image url="IMAGES[N]" align="center"/>` trong đó `IMAGES` là mảng URL từ Phase 1
- Chèn ảnh đúng vị trí tương ứng trong bản gốc (dựa vào image-context map)

### Ảnh nào nên chèn, ảnh nào bỏ?

- **Chèn**: Banner, ảnh minh hoạ nội dung, screenshot, biểu đồ, infographic
- **Có thể bỏ**: Ảnh avatar tác giả, icon trang trí nhỏ, ảnh QR code cuối bài

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

Dùng khi cell chứa list, code block, hoặc nội dung nhiều dòng:

```html
<lark-table column-widths="250,480" header-row="true">
<lark-tr>
<lark-td>

**Header 1**

</lark-td>
<lark-td>

**Header 2**

</lark-td>
</lark-tr>
<lark-tr>
<lark-td>

Row data

</lark-td>
<lark-td>

- List item 1
- List item 2

</lark-td>
</lark-tr>
</lark-table>
```

---

## Chiến lược chia chunk cho bài dài

Khi markdown > 5000 ký tự, chia thành nhiều chunk:

### Chunk 1: `docs +create`

```bash
lark-cli docs +create --title "TIÊU_ĐỀ" --markdown "CHUNK_1_CONTENT"
```

### Chunk 2+: `docs +update --mode append`

```bash
lark-cli docs +update --doc DOC_ID --mode append --markdown "CHUNK_N_CONTENT"
```

### Quy tắc chia chunk

1. **Bẻ tại heading boundary** (`##` hoặc `###`) — không bẻ giữa đoạn
2. **Không bẻ giữa callout/grid đang mở** — mỗi chunk phải có HTML tag đóng đầy đủ
3. **Chunk đầu tiên** nên chứa: metadata header + nội dung đến hết section đầu tiên
4. **Mỗi chunk** ~3000-5000 ký tự markdown
5. **Heredoc cho nội dung có ký tự đặc biệt** — dùng `"$(cat <<'ENDOFMD' ... ENDOFMD)"` hoặc single-quote string

### Xử lý ký tự đặc biệt trong shell

- Single quote `'` trong nội dung: dùng heredoc `$(cat <<'ENDOFMD' ... ENDOFMD)`
- Double quote `"` trong nội dung: dùng single-quote wrapper
- Nếu cả hai đều có: dùng heredoc và tránh pattern `ENDOFMD` trong content

---

## Checklist format trước khi tạo

- [ ] Title không lặp lại trong markdown body
- [ ] Metadata callout ở đầu bài
- [ ] `---` phân cách giữa các section lớn
- [ ] Tất cả `[[IMG_N]]` đã thay bằng `<image url="..."/>`
- [ ] Callout, grid, table đều đóng tag đúng
- [ ] Không có dòng trống thừa liên tiếp (> 2 dòng)
