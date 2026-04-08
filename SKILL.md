---
name: wechat-to-lark
version: 1.0.0
description: |
  Pipeline dịch bài viết WeChat sang tiếng Việt và đăng lên LarkSuite.
  Kích hoạt khi user cung cấp link mp.weixin.qq.com và yêu cầu dịch/clone bài viết.
  Bao gồm: trích xuất nội dung + ảnh, dịch thuần Việt, tạo Lark doc có ảnh, QA đối chiếu.
metadata:
  author: vudan
  updated: 2026-04-08
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

## Khi nào kích hoạt

- User cung cấp URL dạng `mp.weixin.qq.com/s/...` và yêu cầu dịch, clone, hoặc đăng lên Lark
- User nói: "dịch bài này", "clone bài này lên lark", "translate this WeChat article"

## Tiền đề bắt buộc

1. **Load web-access skill** và đảm bảo CDP proxy đang chạy:
   ```bash
   node "${CLAUDE_SKILL_DIR}/../web-access/scripts/check-deps.mjs"
   ```
   Nếu chưa chạy, hướng dẫn user bật Chrome remote debugging.

2. **Load lark-doc skill** (và lark-shared) để sử dụng `lark-cli docs +create/+update`.

3. **Đọc site pattern WeChat** nếu có: `web-access/references/site-patterns/mp.weixin.qq.com.md`

## Pipeline tổng quan

```
┌─────────────────────────────────────────────────────────────────┐
│                    WeChat URL (mp.weixin.qq.com)                │
└──────────────────────────┬──────────────────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│  Phase 1: EXTRACT                                                │
│  CDP proxy → scroll bottom → regex innerHTML                     │
│  Output: ① text với [[IMG_N]] markers  ② mảng URL ảnh           │
└──────────────────────────┬───────────────────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│  Phase 2: MAP                                                    │
│  Xác định context text xung quanh mỗi [[IMG_N]]                 │
│  Output: ③ image-context map (JSON) ← LƯU LẠI CHO PHASE 5      │
└──────────────────────────┬───────────────────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│  Phase 3: TRANSLATE                                              │
│  Dịch thuần Việt, giữ nguyên [[IMG_N]] markers                  │
│  Input: ① text gốc    Output: ④ text tiếng Việt với markers     │
└──────────────────────────┬───────────────────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│  Phase 4: CREATE                                                 │
│  Thay [[IMG_N]] → <image url="②[N]"/>  +  Lark formatting       │
│  Input: ②④    Output: ⑤ Lark doc URL + doc_id                   │
│  Công cụ: lark-cli docs +create / +update --mode append          │
└──────────────────────────┬───────────────────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│  Phase 5: QA                                                     │
│  Fetch ⑤ → đối chiếu với ①②③                                    │
│  Kiểm tra: heading count, image count, vị trí ảnh, đoạn thiếu   │
│  Nếu lỗi → docs +update --mode insert_before/after để sửa       │
└──────────────────────────────────────────────────────────────────┘
```

**Tóm tắt data flow:**

| Output | Sinh ra ở | Dùng lại ở |
|--------|----------|------------|
| ① Text + markers | Phase 1.4 | Phase 3, Phase 5 |
| ② Mảng URL ảnh | Phase 1.3 | Phase 4 |
| ③ Image-context map | Phase 2 | Phase 4, Phase 5 |
| ④ Text Việt + markers | Phase 3 | Phase 4 |
| ⑤ Lark doc_id + URL | Phase 4 | Phase 5 |

---

## Phase 1: EXTRACT — Trích xuất nội dung

> Đọc chi tiết tại [references/wechat-extraction.md](references/wechat-extraction.md)

### 1.1 Mở bài viết

```bash
# Mở URL qua CDP proxy
curl -s "http://localhost:3456/new?url=WECHAT_URL"
# → Lưu targetId trả về

# Kiểm tra trang đã load
curl -s "http://localhost:3456/info?target=TARGET_ID"
# → Lấy title từ trường "title"
```

### 1.2 Scroll để trigger lazy loading

```bash
curl -s "http://localhost:3456/scroll?target=TARGET_ID&direction=bottom"
# Đợi 2 giây cho ảnh load
sleep 2
```

### 1.3 Trích xuất ảnh (QUAN TRỌNG)

**KHÔNG DÙNG `querySelectorAll("img")`** — WeChat trả về 0 kết quả.

Dùng **regex trên innerHTML**:

**QUAN TRỌNG:** Luôn dùng `--data-binary @-` với heredoc thay vì `-d '...'` để tránh lỗi shell escape.

```bash
curl -s -X POST "http://localhost:3456/eval?target=TARGET_ID" --data-binary @- << 'EVALEOF'
(function() {
  var html = document.getElementById("js_content").innerHTML;
  var seen = {};
  var images = [];

  // Pattern 1 (chính): data-src — ảnh lazy-loaded
  var re1 = /data-src="(https?:\/\/mmbiz\.qpic\.cn[^"]+)"/g;
  var m;
  while ((m = re1.exec(html)) !== null) {
    var url = m[1].replace(/&amp;/g, "&");
    if (!seen[url]) { seen[url] = 1; images.push(url.split("#")[0]); }
  }

  // Pattern 2 (phụ): src — ảnh đã load xong, dedup với pattern 1
  var re2 = /\ssrc="(https?:\/\/mmbiz\.qpic\.cn[^"]+)"/g;
  while ((m = re2.exec(html)) !== null) {
    var url = m[1].replace(/&amp;/g, "&");
    var base = url.split("#")[0].split("&tp=")[0];
    if (!seen[base] && !seen[url]) { seen[url] = 1; images.push(url.split("#")[0]); }
  }

  return JSON.stringify({total: images.length, urls: images});
})()
EVALEOF
```

### 1.4 Trích xuất text với image markers

**QUAN TRỌNG:** Dùng `--data-binary @-` với heredoc, KHÔNG dùng `-d '...'` (gây lỗi shell escape với regex).

```bash
curl -s -X POST "http://localhost:3456/eval?target=TARGET_ID" --data-binary @- << 'EVALEOF'
(function() {
  var article = document.getElementById("js_content");
  if (!article) return "NO_CONTENT";
  var html = article.innerHTML;
  var idx = 0;
  var marked = html.replace(/<img[^>]*data-src="(https?:\/\/mmbiz[^"]+)"[^>]*>/g, function() {
    return "[[IMG_" + (idx++) + "]]";
  });
  var text = marked.replace(/<[^>]+>/g, "\n").replace(/\n{3,}/g, "\n\n").trim();
  var lines = text.split("\n").filter(function(l) {
    var t = l.trim();
    if (!t) return false;
    if (/^(重播|分享|赞|关闭|观看更多|更多|退出全屏|继续观看|继续播放|播放|倍速|全屏)/.test(t)) return false;
    if (/^(已关注|关注|转载|写下你的评论|视频详情|点赞|在看|已同步到看一看)/.test(t)) return false;
    if (/^(切换到|进度条|倍速播放中|您的浏览器不支持|0\/0|分享视频)/.test(t)) return false;
    if (/^(0\.5倍|0\.75倍|1\.0倍|1\.5倍|2\.0倍|超清|流畅)$/.test(t)) return false;
    if (/^，时长$/.test(t)) return false;
    if (/^\d{2}:\d{2}$/.test(t)) return false;
    return true;
  });
  return lines.join("\n");
})()
EVALEOF
```

### 1.5 Đóng tab

```bash
curl -s "http://localhost:3456/close?target=TARGET_ID"
```

---

## Phase 2: MAP — Bản đồ vị trí ảnh

> Đọc chi tiết tại [references/wechat-extraction.md](references/wechat-extraction.md)

Với mỗi `[[IMG_N]]` trong text đã trích xuất:
- Lấy 60-100 ký tự trước và sau marker
- Kết quả: `{ img: N, textBefore: "...", textAfter: "..." }`

Dùng **Script 3** trong [references/wechat-extraction.md](references/wechat-extraction.md#script-3-image-context-mapping-bản-đồ-vị-trí-ảnh) — KHÔNG duplicate script ở đây.

**⚠️ Lưu output JSON** từ Phase 2 — sẽ cần lại ở Phase 5 (QA) để kiểm tra vị trí ảnh.

Bản đồ này dùng ở Phase 4 để chèn `<image>` đúng vị trí, và Phase 5 để QA.

---

## Phase 3: TRANSLATE — Dịch sang tiếng Việt

> Đọc chi tiết tại [references/translation-guidelines.md](references/translation-guidelines.md)

### Nguyên tắc cốt lõi

- **Thuần Việt**: Viết như tác giả Việt Nam, không phải dịch máy
- **Giữ nguyên**: Tên sản phẩm, thuật ngữ kỹ thuật, tên Skill, `[[IMG_N]]` markers
- **Adapt**: Ẩn dụ, idiom Trung Quốc → giải thích tự nhiên cho người Việt
- **Format**: Nhận diện nội dung phù hợp cho callout, grid, table của Lark

### Quy trình

1. Dịch theo từng section (heading), giữ nguyên cấu trúc heading
2. Giữ nguyên tất cả `[[IMG_N]]` markers — chúng sẽ được thay thế ở Phase 4
3. Đánh dấu nội dung phù hợp cho callout (insight quan trọng, cảnh báo, trích dẫn)

---

## Phase 4: CREATE — Tạo Lark document

> Đọc chi tiết tại [references/lark-formatting.md](references/lark-formatting.md)

### 4.1 Chuẩn bị Markdown

1. Thêm metadata header (callout với nguồn, tác giả, link gốc)
2. Thay mỗi `[[IMG_N]]` bằng `<image url="IMAGES[N]" align="center"/>`
   - `IMAGES[N]` = URL ảnh thứ N từ Phase 1.3
3. Thêm callout, grid, table theo đánh dấu ở Phase 3
4. Thêm `---` giữa các section lớn

### 4.2 Tạo document

**Bài ngắn** (< 5000 ký tự markdown):
```bash
lark-cli docs +create --title "TITLE" --markdown "FULL_CONTENT"
```

**Bài dài** (chia chunk tại heading boundaries):
```bash
# Chunk 1
lark-cli docs +create --title "TITLE" --markdown "CHUNK_1"
# → Lưu doc_id

# Chunk 2+
lark-cli docs +update --doc DOC_ID --mode append --markdown "CHUNK_2"
```

### 4.3 Lưu ý quan trọng

- URL ảnh WeChat (`mmbiz.qpic.cn`) công khai, dùng trực tiếp trong `<image url="..."/>`
- Không bẻ chunk giữa câu hoặc giữa callout/grid đang mở
- Nếu user chỉ định `--folder-token` hoặc `--wiki-node`, truyền qua cho `docs +create`

---

## Phase 5: QA — Đối chiếu bản dịch

> Đọc chi tiết tại [references/qa-checklist.md](references/qa-checklist.md)

### 5.1 Fetch bản dịch

```bash
lark-cli docs +fetch --doc DOC_ID
```

### 5.2 Đối chiếu

**Cần:** Output JSON từ Phase 2 (image-context map) + text gốc từ Phase 1.4

| Hạng mục | Cách kiểm tra |
|----------|--------------|
| Đủ section | Đếm heading ## / ### gốc vs dịch |
| Đủ ảnh | Đếm `[[IMG_N]]` gốc vs `<image>` trong Lark doc |
| Ảnh đúng vị trí | So sánh context xung quanh ảnh với image-context map (Phase 2) |
| Không thiếu đoạn | So sánh số đoạn văn gốc vs dịch |
| Không còn marker | Kiểm tra không còn `[[IMG_N]]` trong Lark doc |

### 5.3 Báo cáo

Xuất bảng QA cho user. Nếu có vấn đề, dùng `docs +update` để sửa.

---

## Tài liệu tham khảo

| File | Khi nào đọc |
|------|------------|
| [references/wechat-extraction.md](references/wechat-extraction.md) | Phase 1-2: trích xuất nội dung WeChat |
| [references/translation-guidelines.md](references/translation-guidelines.md) | Phase 3: dịch nội dung |
| [references/lark-formatting.md](references/lark-formatting.md) | Phase 4: tạo Lark document |
| [references/qa-checklist.md](references/qa-checklist.md) | Phase 5: QA đối chiếu |
