# Trích xuất nội dung WeChat (mp.weixin.qq.com)

> **⚠️ Quy tắc shell:** Tất cả script eval trong file này phải gọi qua `--data-binary @- << 'EVALEOF'` (heredoc). KHÔNG dùng `-d '...'` — sẽ gây lỗi `{"error":"Uncaught"}` do shell bash interpret sai regex/escape.

## Cấu trúc DOM của bài viết WeChat

| Thành phần | Selector |
|-----------|----------|
| Nội dung bài | `document.getElementById("js_content")` |
| Tiêu đề | `document.getElementById("activity-name")` |
| Tên tác giả/kênh | `document.getElementById("js_name")` |

## Vấn đề cốt lõi: Tại sao DOM selector không lấy được ảnh

```
❌ document.querySelectorAll("#js_content img")  → trả về 0
✅ Regex trên innerHTML                          → trả về 18-20 ảnh
```

**Nguyên nhân:** WeChat render DOM với cấu trúc nested sections đặc biệt, khiến `querySelectorAll` không "nhìn thấy" thẻ `<img>` bên trong `#js_content`. Thêm vào đó, ảnh dùng lazy loading qua thuộc tính `data-src` (không phải `src`).

**Giải pháp:** Parse trực tiếp `innerHTML` bằng regex, tìm pattern `data-src="https://mmbiz.qpic.cn/..."`.

## Bước bắt buộc trước khi trích xuất: Scroll

Ảnh WeChat dùng lazy loading — chỉ load khi nằm trong viewport. **Phải scroll xuống cuối trang trước:**

```bash
curl -s "http://localhost:3456/scroll?target=TARGET_ID&direction=bottom"
sleep 2
```

Đợi 2 giây cho ảnh hoàn tất loading.

---

## Script 1: Trích xuất ảnh (Image Extraction)

```javascript
(function() {
  var html = document.getElementById("js_content").innerHTML;
  var seen = {};
  var images = [];
  
  // Pattern chính: data-src (lazy-loaded images)
  var re1 = /data-src="(https?:\/\/mmbiz\.qpic\.cn[^"]+)"/g;
  var m;
  while ((m = re1.exec(html)) !== null) {
    var url = m[1].replace(/&amp;/g, "&");
    if (!seen[url]) {
      seen[url] = 1;
      images.push(url.split("#")[0]); // bỏ #imgIndex
    }
  }
  
  // Pattern phụ: src (ảnh đã load xong, dedup với data-src)
  var re2 = /\ssrc="(https?:\/\/mmbiz\.qpic\.cn[^"]+)"/g;
  while ((m = re2.exec(html)) !== null) {
    var url = m[1].replace(/&amp;/g, "&");
    var base = url.split("#")[0].split("&tp=")[0];
    if (!seen[base] && !seen[url]) {
      seen[url] = 1;
      images.push(url.split("#")[0]);
    }
  }
  
  return JSON.stringify({total: images.length, urls: images});
})()
```

**Lưu ý:**
- `&amp;` trong HTML phải replace thành `&`
- URL có `#imgIndex=N` — dùng để sắp xếp thứ tự, nhưng bỏ đi khi lưu URL
- Dedup giữa `data-src` và `src` bằng cách strip `&tp=` param

---

## Script 2: Trích xuất text với image markers VÀ format markers

> ⚠️ **BẮT BUỘC dùng phiên bản này** — phiên bản cũ (chỉ strip tags) làm mất hết bold/highlight của bản gốc, dẫn đến bản dịch phẳng lì không trung thành với formatting WeChat.

WeChat thường nhấn mạnh key insight qua:
- `<strong>` / `<b>` → in đậm
- `<span style="font-weight: bold|600|700|...">` → in đậm
- `<span style="color: rgb(...)">` (đặc biệt cam/đỏ) → highlight nhấn mạnh
- `<em>` / `<i>` → nghiêng

Script bên dưới convert tất cả những loại này thành markdown markers (`**bold**`, `*italic*`) **TRƯỚC** khi strip HTML, để Phase 3 (translate) có thể detect và bảo toàn.

```javascript
(function() {
  var article = document.getElementById("js_content");
  if (!article) return "NO_CONTENT";
  var html = article.innerHTML;

  // === STEP 1: Image markers ===
  var idx = 0;
  html = html.replace(/<img[^>]*data-src="(https?:\/\/mmbiz[^"]+)"[^>]*>/g, function() {
    return "[[IMG_" + (idx++) + "]]";
  });

  // === STEP 2: Convert format tags → markdown markers (TRƯỚC khi strip) ===
  // Pass nhiều lần để xử lý nested tags

  function applyFormat(h) {
    var changed = true;
    var iter = 0;
    while (changed && iter < 10) {
      changed = false;
      iter++;

      // Bold: <strong>, <b>
      var h2 = h.replace(/<(strong|b)\b[^>]*>([\s\S]*?)<\/\1>/gi, function(_, tag, inner) {
        var t = inner.trim();
        return t ? " **" + t + "** " : "";
      });
      if (h2 !== h) { h = h2; changed = true; }

      // Bold qua inline-style font-weight: bold | 600 | 700 | 800 | 900
      var h3 = h.replace(/<span\b[^>]*style="[^"]*font-weight:\s*(?:bold|[6-9]00)[^"]*"[^>]*>([\s\S]*?)<\/span>/gi, function(_, inner) {
        var t = inner.trim();
        return t ? " **" + t + "** " : "";
      });
      if (h3 !== h) { h = h3; changed = true; }

      // Highlight qua màu chữ KHÁC mặc định (đen/xám). WeChat dùng cam/đỏ cho emphasis.
      // Loại trừ: black, #000, rgb(0,0,0), rgb gần 0, gray
      var h4 = h.replace(/<span\b[^>]*style="[^"]*color:\s*([^;"]+)[^"]*"[^>]*>([\s\S]*?)<\/span>/gi, function(m, color, inner) {
        var c = color.toLowerCase().replace(/\s+/g, "");
        // Bỏ qua màu mặc định / xám đen
        if (/^(black|#000(000)?|inherit|currentcolor)$/.test(c)) return m.replace(/<[^>]+>/g, "");
        var rgbMatch = c.match(/rgb\((\d+),(\d+),(\d+)\)/);
        if (rgbMatch) {
          var r = +rgbMatch[1], g = +rgbMatch[2], b = +rgbMatch[3];
          // Loại trừ đen + xám gần đen
          if (r < 50 && g < 50 && b < 50) return m.replace(/<[^>]+>/g, "");
        }
        // Còn lại = highlight có chủ ý → mark as bold
        var t = inner.trim();
        return t ? " **" + t + "** " : "";
      });
      if (h4 !== h) { h = h4; changed = true; }

      // Italic: <em>, <i>
      var h5 = h.replace(/<(em|i)\b[^>]*>([\s\S]*?)<\/\1>/gi, function(_, tag, inner) {
        var t = inner.trim();
        return t ? " *" + t + "* " : "";
      });
      if (h5 !== h) { h = h5; changed = true; }
    }
    return h;
  }

  html = applyFormat(html);

  // === STEP 3: Strip các tag còn lại, giữ markers ===
  var text = html.replace(/<[^>]+>/g, "\n").replace(/\n{3,}/g, "\n\n").trim();

  // === STEP 4: Clean up double-bold artifacts như "** **" hoặc "****" ===
  text = text.replace(/\*\*\s*\*\*/g, "");          // empty bold
  text = text.replace(/\*\*([^*]+)\*\*/g, function(_, inner) {
    // Trim space ngay trong bold marker
    return "**" + inner.trim() + "**";
  });
  text = text.replace(/ {2,}/g, " ");

  // === STEP 5: Filter noise từ video player ===
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
```

**Kết quả mẫu:**

```
Hãy nói về Gemini 3.5 Flash trước.
Nhiều người nghe thấy Flash sẽ nghĩ ngay đến "nhanh". Lần này đúng là vẫn nhanh, nhưng điều Google muốn nhấn mạnh đã đi xa hơn một bước:
**Nó sẽ trở thành bộ não thực thi của Agent.**
Để hiểu đúng tầm quan trọng...
**Phiên bản flagship:** GPT-5...
**Phiên bản nhẹ:** GPT-5 mini...
**Lần này Google đã phá vỡ ranh giới đó.**
Gemini 3.5 Flash (phiên bản nhẹ) mới ra, trên một số điểm chuẩn quan trọng đã **vượt mặt Gemini 3.1 Pro** (flagship thế hệ trước)...
[[IMG_1]]
```

Phase 3 sẽ bảo toàn các `**...**` này trong bản dịch — xem `translation-guidelines.md → MANDATORY: format preservation`.

---

## Script 4: Trích xuất video

WeChat embed video qua `<video src="https://mpvideo.qpic.cn/..." data-mpvid="wxv_...">`. Mỗi video có:
- **`data-mpvid`**: ID nội bộ WeChat (`wxv_...`) — dùng để dedup
- **`src`** trong `<video>`: URL `.mp4` thực, có chứa token auth (`auth_key`, `auth_info`) hết hạn theo thời gian
- **`poster`**: ảnh thumbnail cho video

**Bắt buộc scroll bottom trước** — video player được render lazy.

```javascript
(function() {
  var html = document.body.innerHTML;
  var result = {};

  // mpvideo IDs (data-mpvid) — dùng để dedup và đếm video
  var ids = [];
  var seenIds = {};
  var re1 = /data-mpvid="([^"]+)"/g;
  var m;
  while ((m = re1.exec(html)) !== null) {
    if (!seenIds[m[1]]) { seenIds[m[1]] = 1; ids.push(m[1]); }
  }
  result.mpvideo_ids = ids;

  // Direct video src URLs (.mp4)
  var srcs = [];
  var re2 = /<video[^>]*src="([^"]+\.mp4[^"]*)"/g;
  while ((m = re2.exec(html)) !== null) {
    srcs.push(m[1].replace(/&amp;/g, "&"));
  }
  result.urls = srcs;

  // Poster thumbnails (theo thứ tự video)
  var posters = [];
  var re3 = /poster="([^"]+)"/g;
  while ((m = re3.exec(html)) !== null) {
    posters.push(m[1].replace(/&amp;/g, "&"));
  }
  result.posters = posters;

  result.total = srcs.length;
  return JSON.stringify(result);
})()
```

**Kết quả mẫu:**
```json
{
  "total": 8,
  "mpvideo_ids": ["wxv_4523809439451381761", "wxv_4523794643523895298", ...],
  "urls": ["https://mpvideo.qpic.cn/0bc3bq.../...mp4?...auth_key=...&vid=wxv_...", ...],
  "posters": ["http://mmbiz.qpic.cn/.../0?wx_fmt=jpeg", ...]
}
```

**Lưu ý:**
- URL video có `auth_key`/`auth_info` — chỉ hợp lệ vài giờ. Tải về càng sớm càng tốt.
- `&amp;` trong URL phải decode → `&` trước khi `curl`.
- Số `urls` có thể nhỏ hơn `mpvideo_ids` nếu vài video không có direct src (chỉ embed iframe).
- Một số video rất dài (full keynote >1h) có thể >500MB — luôn HEAD check trước:
  ```bash
  curl -sI -L "$URL" --max-time 30 | grep -i content-length
  ```

---

## Script 3: Image-Context Mapping (bản đồ vị trí ảnh)

```javascript
(function() {
  var article = document.getElementById("js_content");
  var html = article.innerHTML;
  
  var idx = 0;
  var marked = html.replace(/<img[^>]*data-src="(https?:\/\/mmbiz[^"]+)"[^>]*>/g, function() {
    return "[[IMG_" + (idx++) + "]]";
  });
  
  // Strip HTML nhưng giữ markers
  var text = marked.replace(/<[^>]+>/g, " ").replace(/\s+/g, " ").trim();
  
  var results = [];
  for (var i = 0; i < idx; i++) {
    var marker = "[[IMG_" + i + "]]";
    var pos = text.indexOf(marker);
    if (pos >= 0) {
      var before = text.substring(Math.max(0, pos - 100), pos).trim();
      var after = text.substring(pos + marker.length, pos + marker.length + 100).trim();
      results.push({
        img: i,
        textBefore: before.slice(-60),
        textAfter: after.slice(0, 60)
      });
    }
  }
  return JSON.stringify(results);
})()
```

**Kết quả mẫu:**
```json
[
  {"img": 0, "textBefore": "", "textAfter": "前几天体验完 DuMate 之后，正好录了一个"},
  {"img": 1, "textBefore": "对于很多刚想要成为 OPC 的人而言，同步给大家。", "textAfter": "在录制开始前，我提前就打了一句话"},
  ...
]
```

Dùng `textBefore` và `textAfter` để xác định chính xác vị trí chèn `<image>` trong bản dịch.

---

## Vấn đề thường gặp

| Vấn đề | Nguyên nhân | Giải pháp |
|--------|-------------|-----------|
| Ảnh trả về 0 | Chưa scroll, hoặc dùng DOM selector | Scroll bottom + dùng regex |
| URL ảnh bị lỗi | `&amp;` chưa decode | Replace `&amp;` → `&` |
| Thiếu ảnh cuối bài | Scroll chưa đủ | Scroll bottom, đợi 2-3s |
| Text có noise video | Video player elements | Dùng noise filter regex |
| Trang yêu cầu verify | WeChat anti-bot | Dùng CDP (Chrome có login) thay vì WebFetch |
| `{"error":"Uncaught"}` khi eval | Shell escape hỏng regex trong `-d '...'` | Dùng `--data-binary @- << 'EVALEOF'` heredoc |
