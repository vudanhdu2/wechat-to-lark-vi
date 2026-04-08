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

## Script 2: Trích xuất text với image markers

```javascript
(function() {
  var article = document.getElementById("js_content");
  var html = article.innerHTML;
  
  // Thay mỗi <img> có data-src bằng [[IMG_N]]
  var idx = 0;
  var marked = html.replace(/<img[^>]*data-src="(https?:\/\/mmbiz[^"]+)"[^>]*>/g, function() {
    return "[[IMG_" + (idx++) + "]]";
  });
  
  // Strip HTML tags
  var text = marked.replace(/<[^>]+>/g, "\n").replace(/\n{3,}/g, "\n\n").trim();
  
  // Lọc noise từ video player WeChat
  var lines = text.split("\n").filter(function(l) {
    var t = l.trim();
    if (!t) return false;
    
    // Video player noise patterns
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

**Kết quả:** Text sạch với markers `[[IMG_0]]`, `[[IMG_1]]`, ... đánh dấu vị trí ảnh.

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
