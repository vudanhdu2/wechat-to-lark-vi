# Tải và chèn video vào Lark document

> Dùng ở **Phase 4b** sau khi đã tạo doc text và chèn ảnh xong.

## Tổng quan

WeChat video không thể link trực tiếp (URL có auth token hết hạn ~vài giờ). Phải:
1. Tải video về file local (`.mp4`)
2. Upload lên Lark qua `+media-insert` — Lark lưu vào CDN nội bộ và render inline player.

---

## Bước 1: Kiểm tra kích thước

WeChat keynote/event videos có thể rất lớn (>500MB). Luôn HEAD-check trước khi tải:

```bash
for url in "${VIDEO_URLS[@]}"; do
  size=$(curl -sI -L "$url" --max-time 30 2>/dev/null | grep -i "content-length" | tail -1 | awk '{print $2}' | tr -d '\r\n')
  mb=$(echo "scale=2; $size / 1048576" | bc 2>/dev/null)
  echo "${mb} MB"
done
```

**Nếu có video > 500MB:** dùng `AskUserQuestion` để hỏi user:
- Tải tất cả (mất nhiều thời gian)
- Bỏ qua video lớn, chỉ chèn link/text mô tả
- Chỉ chèn link cho video lớn, tải bình thường các video nhỏ

---

## Bước 2: Download song song

```bash
mkdir -p /tmp/wechat_videos && cd /tmp/wechat_videos

i=1
for url in "${VIDEO_URLS[@]}"; do
  curl -s -L "$url" -o "video_${i}.mp4" --max-time 600 &
  i=$((i+1))
done
wait

ls -la *.mp4
```

**Lưu ý:**
- URL từ WeChat có thể chứa `&amp;` — phải decode → `&` trước khi `curl`
- `--max-time 600` (10 phút) đủ cho video <1GB
- Chạy song song với `&` + `wait` — nhanh hơn nhiều so với tuần tự

---

## Bước 3: Insert vào Lark

⚠️ **Bắt buộc `cd` vào thư mục chứa file.** lark-cli reject absolute path (đặc biệt Windows `C:/...`).

### Case A — Có anchor text rõ ràng trong bản dịch

```bash
cd /tmp/wechat_videos

lark-cli docs +media-insert \
  --doc DOC_ID \
  --type file \
  --file ./video_3.mp4 \
  --file-view preview \
  --selection-with-ellipsis "unique substring khớp duy nhất 1 paragraph"
```

**`--selection-with-ellipsis`:**
- Match text trong block content (case-insensitive substring)
- Nếu text xuất hiện nhiều nơi: dùng cú pháp `"unique start...unique end"`
- Video được chèn AFTER block khớp (ở top-level)

### Case B — Không có anchor → thêm caption trước

```bash
# B1: Thêm caption paragraph trước (block_insert_after sau text block trước đó)
echo '<p><b>📺 Demo: [mô tả ngắn]:</b></p>' | \
  lark-cli docs +update \
    --api-version v2 \
    --doc DOC_ID \
    --command block_insert_after \
    --block-id PREVIOUS_TEXT_BLOCK_ID \
    --doc-format xml \
    --content -

# B2: Insert video, dùng chính caption làm anchor
lark-cli docs +media-insert \
  --doc DOC_ID \
  --type file \
  --file ./video_X.mp4 \
  --file-view preview \
  --selection-with-ellipsis "📺 Demo: [mô tả ngắn]"
```

### Case C — Chèn video TRƯỚC một paragraph cụ thể

Dùng khi video phải nằm ngay trước một section/paragraph có sẵn:

```bash
lark-cli docs +media-insert \
  --doc DOC_ID \
  --type file \
  --file ./video_4.mp4 \
  --file-view preview \
  --before \
  --selection-with-ellipsis "Thứ hai, chỉnh sửa hội thoại"
```

### Case D — Nhiều video liền nhau (không thể chèn caption giữa)

⚠️ **`block_insert_after` trên file/video block sẽ FAIL** với:
```
degrade_code=4000020, msg=Block schema validation failed
```

Cách xử lý:
1. Insert video A trước với anchor X
2. Insert video B với anchor là paragraph TIẾP THEO + `--before` flag
3. Nếu cần caption giữa A và B: xoá A, chèn caption, chèn A lại, chèn B

Hoặc đơn giản: thiết kế bản dịch sao cho caption text đã sẵn TRƯỚC mỗi video.

---

## Tham số `--file-view`

| Giá trị | Hiệu ứng |
|---------|----------|
| `card` (default) | Thumbnail card — user click mới xem |
| `preview` | **Inline player** — video render trực tiếp trong doc (CẦN cho video) |
| `inline` | Inline link (không phù hợp video) |

Luôn dùng `--file-view preview` cho video.

---

## Multipart upload tự động

File >20MB → lark-cli tự động dùng multipart upload (4MB/chunk):
```
  Block 1/196 uploaded (4.0 MB)
  Block 2/196 uploaded (4.0 MB)
  ...
  Block 196/196 uploaded (3.6 MB)
File uploaded: L4wzbI7OhoUUrcxQwfCl5lxigyd
Binding uploaded media to block doxlgxiEQYg5i8MKQdofOp6QoGc
```

File 800MB cần ~5-10 phút tuỳ băng thông. KHÔNG ngắt tiến trình.

---

## Kết quả: cấu trúc XML của video block

Sau khi insert, fetch doc với `--detail full` sẽ thấy:

```xml
<figure id="doxlg..." view-type="Preview">
  <source id="doxlg..." href="https://internal-api-drive-stream-sg.larksuite.com/..."
          file-token="L4wz..." mime="video/mp4" .../>
</figure>
```

QA dùng pattern `<figure[^>]*view-type="Preview"` để đếm video. KHÔNG dùng `<video>` hay `<file>` — Lark không dùng những tag này.

---

## Dọn dẹp sau khi xong

```bash
cd / && rm -rf /tmp/wechat_videos
```

Video đã được upload lên Lark CDN, không cần giữ bản local nữa.

---

## Troubleshooting

| Lỗi | Nguyên nhân | Cách sửa |
|-----|-------------|----------|
| `unsafe file path: --file must be a relative path` | Dùng absolute path | `cd` vào thư mục, dùng `./filename` |
| `degrade_code=4000020, Block schema validation failed` | `block_insert_after` trên file block | Chèn captions TRƯỚC khi insert video, hoặc dùng `--before` với paragraph kế tiếp |
| `degrade_code=1011, Instruction produced no document changes` | Pattern str_replace không khớp, hoặc content giống hệt | Kiểm tra escape regex (`\\.` cho dấu chấm) |
| Video URL trả về 403 | Auth token hết hạn | Mở lại bài WeChat, lấy URL mới ngay, tải liền |
| Upload hang > 15 phút | Băng thông kém với file lớn | Đảm bảo network ổn định; có thể chạy lại — Lark dedup theo file content |
