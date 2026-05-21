---
name: wechat-to-lark
version: 2.0.0
description: |
  Pipeline dịch bài viết WeChat sang tiếng Việt và đăng lên LarkSuite.
  Kích hoạt khi user cung cấp link mp.weixin.qq.com và yêu cầu dịch/clone bài viết.
  Bao gồm: trích xuất nội dung + ảnh + video, dịch thuần Việt, tạo Lark doc có ảnh & video embed, QA đối chiếu.
metadata:
  author: vudan
  updated: 2026-05-21
  requires:
    skills: ["web-access", "lark-doc"]
    bins: ["lark-cli"]
---

# wechat-to-lark Skill

## ⚠️ Quy tắc shell quan trọng

Khi gọi `/eval` qua CDP proxy, **LUÔN** dùng heredoc:
```bash
curl -s -X POST "http://localhost:3456/eval?target=ID" --data-binary @- << 'EVALEOF'
// JavaScript code here
EVALEOF
```
**KHÔNG BAO GIỜ** dùng `-d '...'` với inline JavaScript — regex và ký tự đặc biệt sẽ bị shell bash interpret sai, gây lỗi `{"error":"Uncaught"}`.

## ⚠️ Quy tắc lark-cli v2 quan trọng

1. **Luôn dùng `--api-version v2`** — v1 đã deprecated.
2. **Cú pháp v2:** `--content --doc-format markdown|xml` (KHÔNG dùng `--markdown` của v1).
3. **`<image url="..."/>` trong markdown content KHÔNG hoạt động ở v2** — image tags sẽ bị strip silently khi tạo doc. Bắt buộc chèn ảnh ở **Phase 4a** bằng XML `<img>` qua `block_insert_after`.
4. **`--file` yêu cầu relative path** trong thư mục hiện tại — `cd` vào thư mục chứa file trước khi chạy `+media-insert`. Path tuyệt đối kiểu Windows (`C:/...`) sẽ bị reject.
5. **`/new` API từ v2.5.3:** dùng `curl -X POST --data-raw "URL" "http://localhost:3456/new"`, không phải query string.

## Khi nào kích hoạt

- User cung cấp URL dạng `mp.weixin.qq.com/s/...` và yêu cầu dịch, clone, hoặc đăng lên Lark
- User nói: "dịch bài này", "clone bài này lên lark", "translate this WeChat article"

## Tiền đề bắt buộc

1. **Load web-access skill** và đảm bảo CDP proxy đang chạy:
   ```bash
   node "${CLAUDE_SKILL_DIR}/../web-access/scripts/check-deps.mjs"
   ```
   Nếu chưa chạy, hướng dẫn user bật Chrome remote debugging.

2. **Load lark-doc skill** (và lark-shared) để sử dụng `lark-cli docs +create/+update/+media-insert`.

3. **Đọc site pattern WeChat** nếu có: `web-access/references/site-patterns/mp.weixin.qq.com.md`

## Pipeline tổng quan

```
┌──────────────────────── WeChat URL ───────────────────────┐
                              ▼
  Phase 1: EXTRACT   text + [[IMG_N]] markers + ảnh URLs + video URLs
                              ▼
  Phase 2: MAP       image-context map + video-context map
                              ▼
  Phase 3: TRANSLATE bản tiếng Việt (giữ [[IMG_N]], đánh dấu video positions)
                              ▼
  Phase 4: CREATE    Lark doc text-only (v2 API, markdown) — KHÔNG ảnh/video
                              ▼
  Phase 4a: IMAGES   fetch block IDs → block_insert_after <img> (XML)
                              ▼
  Phase 4b: VIDEOS   download mp4 → +media-insert --file-view preview
                              ▼
  Phase 5: QA        đối chiếu: <img>, <figure view-type="Preview">, headings
```

**Data flow:**

| Output | Sinh ra ở | Dùng lại ở |
|--------|----------|------------|
| ① Text + `[[IMG_N]]` markers | Phase 1.5 | Phase 3, Phase 5 |
| ② Mảng URL ảnh | Phase 1.3 | Phase 4a |
| ③ Mảng URL video | Phase 1.4 | Phase 4b |
| ④ Image-context map | Phase 2 | Phase 4a, Phase 5 |
| ⑤ Video-context map | Phase 2 | Phase 4b |
| ⑥ Text Việt | Phase 3 | Phase 4 |
| ⑦ Lark doc_id + URL | Phase 4 | Phase 4a, 4b, 5 |
| ⑧ Block ID mapping | Phase 4a | Phase 4a |

---

## Phase 1: EXTRACT — Trích xuất nội dung

> Đọc chi tiết tại [references/wechat-extraction.md](references/wechat-extraction.md)

### 1.1 Mở bài viết

```bash
# v2.5.3+: POST với URL trong body
curl -s -X POST --data-raw "WECHAT_URL" "http://localhost:3456/new"
# → Lưu targetId trả về

# Kiểm tra trang đã load (đợi 2-3s cho WeChat verify nếu cần)
sleep 3
curl -s "http://localhost:3456/info?target=TARGET_ID"
# → Lấy title từ trường "title"
```

### 1.2 Scroll để trigger lazy loading

```bash
curl -s "http://localhost:3456/scroll?target=TARGET_ID&direction=bottom"
sleep 3  # đợi ảnh + video metadata load
```

### 1.3 Trích xuất ảnh (Script 1)

**KHÔNG DÙNG `querySelectorAll("img")`** — WeChat trả về 0 kết quả. Dùng regex trên innerHTML.

Xem [references/wechat-extraction.md → Script 1](references/wechat-extraction.md#script-1-trích-xuất-ảnh).

### 1.4 Trích xuất video (Script 4 — MỚI)

WeChat embed video qua `<video src="...mpvideo.qpic.cn..." data-mpvid="wxv_...">`. Bắt buộc scroll bottom trước.

Xem [references/wechat-extraction.md → Script 4](references/wechat-extraction.md#script-4-trích-xuất-video).

### 1.5 Trích xuất text với image markers (Script 2)

Xem [references/wechat-extraction.md → Script 2](references/wechat-extraction.md#script-2-trích-xuất-text).

### 1.6 Đóng tab

```bash
curl -s "http://localhost:3456/close?target=TARGET_ID"
```

---

## Phase 2: MAP — Bản đồ vị trí ảnh & video

> Đọc chi tiết tại [references/wechat-extraction.md](references/wechat-extraction.md)

**Image-context map** — với mỗi `[[IMG_N]]`:
- Lấy 60-100 ký tự trước & sau marker
- Dùng để chèn ảnh đúng vị trí ở Phase 4a

**Video-context map** — với mỗi video URL:
- Trong text gốc, tìm cụm 视频 / 看视频 / "完整中英文双语视频" hoặc placeholder text gần đoạn video
- Ghi nhận unique snippet (50-80 ký tự) làm anchor cho `--selection-with-ellipsis` ở Phase 4b
- Nếu video không có anchor text rõ ràng → chuẩn bị caption riêng (vd: "📺 Demo: [mô tả ngắn]:")

Dùng Script 3 trong [references/wechat-extraction.md](references/wechat-extraction.md#script-3-image-context-mapping).

**⚠️ Lưu JSON Phase 2** — cần ở Phase 4a, 4b và Phase 5.

---

## Phase 3: TRANSLATE — Dịch sang tiếng Việt

> Đọc chi tiết tại [references/translation-guidelines.md](references/translation-guidelines.md)

### Nguyên tắc cốt lõi

- **Thuần Việt**: Viết như tác giả Việt Nam, không phải dịch máy
- **Giữ nguyên**: Tên sản phẩm, thuật ngữ kỹ thuật, tên Skill, `[[IMG_N]]` markers
- **Adapt**: Ẩn dụ, idiom Trung Quốc → giải thích tự nhiên cho người Việt
- **Format**: Nhận diện nội dung phù hợp cho callout, grid, table của Lark

### Xử lý vị trí video

- Nếu đoạn gốc có text giới thiệu video → dịch text đó, dùng làm anchor cho video sau này
- Nếu không có anchor rõ → trong bản dịch, chèn dòng caption `**📺 [Mô tả ngắn về video]:**` ở vị trí phù hợp; caption này sẽ là anchor cho `--selection-with-ellipsis`

---

## Phase 4: CREATE — Tạo Lark document text-only

> Đọc chi tiết tại [references/lark-formatting.md](references/lark-formatting.md)

⚠️ **Phase này CHỈ tạo text + headings + lists + callouts.** Ảnh và video sẽ được chèn ở Phase 4a/4b. KHÔNG nhúng `<image>` hay `<img>` trong markdown content — sẽ bị strip silently.

### 4.1 Chuẩn bị Markdown

1. Thêm metadata header (blockquote hoặc callout với nguồn, tác giả, link gốc)
2. **Xoá các `[[IMG_N]]` markers** trong text dịch — sẽ chèn lại ở Phase 4a
3. **Giữ lại các caption video** (vd: `**📺 Demo: ...:**`) làm anchor cho Phase 4b
4. Thêm callout, grid, table theo Phase 3
5. Thêm `---` giữa các section lớn

### 4.2 Ghi file markdown (relative path bắt buộc)

```bash
cat > ./article_vi.md << 'MDEOF'
[FULL MARKDOWN CONTENT]
MDEOF
```

### 4.3 Tạo document (v2 API)

```bash
lark-cli docs +create \
  --api-version v2 \
  --title "TITLE" \
  --doc-format markdown \
  --content "@./article_vi.md"
# → Lưu document_id từ data.document.document_id
```

**Tuỳ chọn:** `--parent-token TOKEN` để đặt vào folder/wiki cụ thể.

---

## Phase 4a: INSERT IMAGES — Chèn ảnh qua block_insert_after

> Đọc chi tiết tại [references/lark-formatting.md → Image insertion](references/lark-formatting.md#chèn-ảnh-qua-block_insert_after-v2)

### 4a.1 Fetch block IDs

```bash
lark-cli docs +fetch \
  --api-version v2 \
  --doc DOC_ID \
  --detail with-ids
# → XML có id="..." trên mỗi block <p>, <h2>, <ul>, etc.
```

### 4a.2 Xác định anchor block cho mỗi ảnh

Với mỗi item trong image-context map (Phase 2):
- Tìm trong XML đã fetch một block có text khớp với `textBefore` (sau khi dịch)
- Lưu `id="..."` của block đó làm anchor

### 4a.3 Insert mỗi ảnh

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

**Cẩn thận thứ tự khi nhiều ảnh dùng cùng anchor:** mỗi insert sau cùng một block_id sẽ đẩy block cũ xuống — thứ tự kết quả bị **đảo ngược**. Hoặc:
- Dùng anchor khác nhau cho mỗi ảnh, HOẶC
- Insert ảnh theo thứ tự ngược trong document, HOẶC
- Sau khi insert ảnh đầu, dùng ảnh đầu (vừa nhận block_id mới) làm anchor cho ảnh kế tiếp

---

## Phase 4b: INSERT VIDEOS — Tải + chèn video

> Đọc chi tiết tại [references/video-handling.md](references/video-handling.md)

### 4b.1 Kiểm tra kích thước trước

```bash
for url in "${VIDEO_URLS[@]}"; do
  curl -sI -L "$url" --max-time 30 | grep -i "content-length"
done
```

**Nếu có video > 500MB** (vd: bản keynote dài >1h): hỏi user qua AskUserQuestion trước khi tải:
- Tải toàn bộ (sẽ mất nhiều thời gian, dùng multipart upload tự động cho Lark)
- Bỏ qua video lớn, chỉ chèn link
- Chỉ chèn link cho video lớn, tải bình thường các video nhỏ

### 4b.2 Download videos (song song)

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

### 4b.3 Insert vào Lark (relative path bắt buộc)

⚠️ **Phải `cd` vào thư mục chứa file** vì lark-cli không nhận absolute path Windows.

**Case A — Video có anchor text trong bản dịch:**

```bash
cd /tmp/wechat_videos
lark-cli docs +media-insert \
  --doc DOC_ID \
  --type file \
  --file ./video_3.mp4 \
  --file-view preview \
  --selection-with-ellipsis "unique substring từ paragraph anchor"
```

**Case B — Video không có anchor text → tự thêm caption trước:**

```bash
# Bước 1: Thêm caption paragraph (block_insert_after sau text block trước đó)
echo '<p><b>📺 [Mô tả video]:</b></p>' | \
  lark-cli docs +update --api-version v2 --doc DOC_ID \
    --command block_insert_after \
    --block-id PREVIOUS_TEXT_BLOCK_ID \
    --doc-format xml --content -

# Bước 2: Insert video, dùng chính caption làm anchor
lark-cli docs +media-insert \
  --doc DOC_ID --type file --file ./video_X.mp4 \
  --file-view preview \
  --selection-with-ellipsis "📺 [Mô tả video]"
```

**Case C — Cần chèn video TRƯỚC một paragraph:**

```bash
lark-cli docs +media-insert \
  --doc DOC_ID --type file --file ./video_4.mp4 \
  --file-view preview \
  --before \
  --selection-with-ellipsis "paragraph text ngay sau vị trí video"
```

### 4b.4 Lưu ý quan trọng

- `--file-view preview` → inline player (CẦN cho video); `card` (default) → chỉ thumbnail
- **Sau khi insert file/video block, KHÔNG thể `block_insert_after` file block đó** — schema validation fail (`degrade_code=4000020`). Thêm captions TRƯỚC khi insert video.
- File >20MB tự động dùng multipart upload (4MB/chunk) — kiên nhẫn chờ; file 800MB ~ vài phút.
- Sau insert xong, **dọn file**: `rm -rf /tmp/wechat_videos`

---

## Phase 5: QA — Đối chiếu bản dịch

> Đọc chi tiết tại [references/qa-checklist.md](references/qa-checklist.md)

### 5.1 Fetch bản dịch

```bash
lark-cli docs +fetch --api-version v2 --doc DOC_ID --doc-format xml
```

### 5.2 Đếm và đối chiếu (script Python)

```python
import re, json, sys
data = json.load(sys.stdin)
content = data['data']['document']['content']

imgs    = re.findall(r'<img[^>]+>', content)
videos  = re.findall(r'<figure[^>]*view-type="Preview"', content)
h2s     = re.findall(r'<h2[^>]*>([^<]+)</h2>', content)
markers = re.findall(r'\[\[IMG_\d+\]\]', content)

print(f'Images: {len(imgs)}   Videos: {len(videos)}')
print(f'H2 sections: {len(h2s)}')
print(f'Markers leftover: {len(markers)} (phải = 0)')
```

### 5.3 Bảng kiểm

| Hạng mục | Cách kiểm | Ngưỡng |
|----------|----------|--------|
| Số ảnh | `<img>` count | = Phase 1.3 total |
| Số video | `<figure view-type="Preview">` count | = Phase 1.4 total |
| Heading H2 | `<h2>` count | khớp gốc |
| Heading H3 | `<h3>` count | khớp gốc (±2 do tổ chức lại) |
| Marker còn sót | `[[IMG_` count | = 0 |
| Vị trí ảnh | textBefore/After khớp Phase 2 | đúng ngữ cảnh |

### 5.4 Báo cáo cho user

Xuất bảng markdown + link tài liệu Lark. Nếu có vấn đề, dùng `docs +update --command block_insert_after/block_delete/str_replace` để sửa.

---

## Tài liệu tham khảo

| File | Khi nào đọc |
|------|------------|
| [references/wechat-extraction.md](references/wechat-extraction.md) | Phase 1-2: trích xuất nội dung WeChat (text/ảnh/video) |
| [references/translation-guidelines.md](references/translation-guidelines.md) | Phase 3: dịch nội dung |
| [references/lark-formatting.md](references/lark-formatting.md) | Phase 4 + 4a: tạo Lark doc + chèn ảnh qua XML |
| [references/video-handling.md](references/video-handling.md) | Phase 4b: download + insert video |
| [references/qa-checklist.md](references/qa-checklist.md) | Phase 5: QA đối chiếu |
