# Hướng dẫn dịch thuần Việt

## 🔴 MANDATORY: Thuần Việt — KHÔNG được dịch sát chữ

**Đây là rào chắn chất lượng số 1.** Khi dịch xong PHẢI scan bản dịch theo bảng anti-pattern và rewrite những chỗ vướng.

> ⚠️ **BẮT BUỘC ĐỌC** [translation-anti-patterns.md](translation-anti-patterns.md) — bảng tra cứu cụ thể những calque/cấu trúc Trung đi vào bản dịch.

### Quy tắc 4 pass trong Phase 3

| Pass | Việc làm | Tiêu chí pass |
|------|----------|--------------|
| **Pass 1: Dịch nghĩa** | Dịch sát ý, giữ markers `**bold**` + `[[IMG_N]]` | Mọi ý gốc đều có trong bản dịch |
| **Pass 2: Rewrite anti-patterns** | Scan bảng mục 1-5 của anti-patterns.md, sửa từng calque | Không còn cụm nào trong cột "❌ Sai" |
| **Pass 3: Glossary consistency** | Term ở mục 6 dùng nhất quán xuyên bài | 1 term = 1 cách dịch trong toàn bài |
| **Pass 4: Đọc to test** | Đọc to 3 đoạn ngẫu nhiên | Không vấp, không "nghe lạ", không câu dài >40 từ |

**KHÔNG bỏ qua Pass 2.** Đây là pass cải thiện chất lượng nhiều nhất.

### Smell test — câu hỏi tự kiểm sau dịch

Đọc lại từng đoạn và hỏi 5 câu:

1. ❓ Có cụm nào nằm trong bảng "❌ Sai" của anti-patterns.md không?
2. ❓ Đọc to lên có vấp chỗ nào không?
3. ❓ Một người Việt thực tế (không phải dịch giả) có viết câu này tự nhiên không?
4. ❓ Có Hán Việt nào thay được bằng thuần Việt không?
5. ❓ Câu nào dài >40 từ không có dấu phẩy/chấm để thở?

Bất kỳ "Có" nào → rewrite đoạn đó.

### Ví dụ pass/fail trực tiếp

```
GỐC TIẾNG TRUNG:
新发的 Gemini 3.5 Flash（轻量版），在好几项关键跑分上反超了今年 2 月才发的 Gemini 3.1 Pro。

❌ DỊCH SÁT CHỮ (FAIL):
Gemini 3.5 Flash mới (phiên bản nhẹ), trên mấy điểm chuẩn then chốt đã vượt ngược Gemini 3.1 Pro phát hành tháng 2 năm nay.

   Vấn đề: "vượt ngược" là calque, "trên mấy điểm chuẩn then chốt" trật tự bổ ngữ kiểu Trung

✅ DỊCH THUẦN VIỆT (PASS):
Gemini 3.5 Flash (phiên bản nhẹ) mới ra, trên một số điểm chuẩn quan trọng đã **vượt mặt Gemini 3.1 Pro** (flagship thế hệ trước vừa ra tháng 2 năm nay).
```

```
GỐC:
能用轻量版就用轻量版，省钱省时间；非要用旗舰版的难题，咬咬牙也得用。

❌ DỊCH SÁT (FAIL):
Việc nào dùng được bản nhẹ thì dùng bản nhẹ, tiết kiệm tiền tiết kiệm thời gian; bài khó thực sự cần bản flagship, cắn răng cũng phải dùng.

   Vấn đề: "Việc nào... thì..." nặng, "bài khó thực sự cần" trật tự Trung, "cắn răng" thiếu kết hợp

✅ DỊCH PASS:
Chiến lược của người dùng và doanh nghiệp cũng rất đơn giản: dùng được phiên bản nhẹ thì dùng, tiết kiệm tiền và thời gian; bài toán thực sự cần flagship thì mới bấm bụng chi tiền.
```

---

## 🔴 MANDATORY: Bảo toàn formatting (bold/italic/highlight)

**Đây là quy tắc CỨNG, không phải gợi ý.** Nếu không tuân thủ, bản dịch sẽ phẳng lì, mất hết nhấn mạnh của tác giả gốc và chất lượng giảm rõ rệt.

### Quy tắc 1 — Mọi `**...**` trong text đã extract đều PHẢI giữ trong bản dịch

Phase 1 (Script 2) đã convert tất cả `<strong>`, `<b>`, `<span style="font-weight:bold">`, và `<span style="color:cam/đỏ">` của WeChat sang markdown bold `**...**`. Trong Phase 3, bạn PHẢI:

1. **Đếm số `**...**` markers** trong text gốc trước khi dịch
2. **Đảm bảo bản dịch có CÙNG SỐ LƯỢNG `**...**`** ở các vị trí tương đương
3. **KHÔNG bỏ markers** kể cả khi dịch rút gọn câu
4. **KHÔNG thêm markers** ở chỗ gốc không có (giữ trung thành với ý đồ tác giả)

### Quy tắc 2 — Áp dụng tương tự cho `*italic*`

Markers `*xxx*` (italic) cũng phải giữ nguyên trong bản dịch.

### Ví dụ ĐÚNG vs SAI

```
GỐC (sau extract):
**Nó sẽ trở thành bộ não thực thi của Agent.**
Để hiểu đúng tầm quan trọng của Flash lần này, cần nắm bối cảnh:
**Phiên bản flagship:** GPT-5 của OpenAI, Claude Opus của Anthropic...

✅ DỊCH ĐÚNG:
**Nó sẽ trở thành bộ não thực thi của Agent.**
Để hiểu đúng tầm quan trọng của Flash lần này, cần nắm bối cảnh:
**Phiên bản flagship:** GPT-5 của OpenAI, Claude Opus của Anthropic...

❌ DỊCH SAI (mất bold):
Nó sẽ trở thành bộ não thực thi của Agent.
Để hiểu đúng tầm quan trọng của Flash lần này, cần nắm bối cảnh:
Phiên bản flagship: GPT-5 của OpenAI, Claude Opus của Anthropic...

❌ DỊCH SAI (thêm bold không có trong gốc):
**Nó sẽ trở thành bộ não thực thi của Agent.**
Để hiểu đúng tầm quan trọng của **Flash lần này**, cần nắm bối cảnh:
**Phiên bản flagship:** GPT-5 của OpenAI, **Claude Opus của Anthropic**...
```

### Quy tắc 3 — Vị trí bold trong câu

Nếu chỉ một phần câu được bold (vd: `Gemini 3.5 Flash đã **vượt mặt Gemini 3.1 Pro** ở...`), giữ bold đúng cụm từ tương đương trong tiếng Việt. KHÔNG dịch xong rồi bôi đen cả câu hay quên bôi đen.

### Self-check trước khi xuất bản dịch

Trước khi đưa cho `lark-cli docs +create`, hãy chạy mental check:
- [ ] Số `**` trong bản dịch = số `**` trong text gốc (chia 2 = số cụm bold)
- [ ] Mỗi cụm bold tiếng Trung gốc → có cụm bold tương ứng tiếng Việt
- [ ] Không có `**` lẻ (mở mà không đóng)

---

## Nguyên tắc tổng quát

Mục tiêu: **Người đọc Việt Nam cảm thấy bài viết được viết bởi một tác giả Việt**, không phải bản dịch từ tiếng Trung.

- Ưu tiên câu ngắn, mạch lạc, rõ ý
- Tránh cấu trúc câu kiểu Trung Quốc (đặt mệnh đề phụ dài trước chủ ngữ)
- Tránh dùng từ Hán-Việt khi có từ thuần Việt tương đương
- Giữ nhịp đọc tự nhiên — đọc to lên phải nghe trôi chảy

## Giữ nguyên (KHÔNG dịch)

| Loại | Ví dụ |
|------|-------|
| Tên sản phẩm / thương hiệu | DuMate, Claude, Cursor, Lark, Notion |
| Thuật ngữ kỹ thuật tiếng Anh | AI, API, Markdown, SaaS, PDF, PPT, Excel |
| Tên Skill / tính năng | ai-notes-for-video, skill-creator, WebSearch |
| Tên riêng người | Lazar, @张咋啦 |
| URL, link | Giữ nguyên |
| `[[IMG_N]]` markers | **Tuyệt đối không sửa, không xóa** |

## Cần chuyển đổi / giải thích

### Ẩn dụ và thành ngữ Trung Quốc

| Gốc | Cách xử lý |
|-----|-----------|
| 降维打击 (đánh chiều không gian thấp hơn) | Giải thích: "Giống kiểu đánh chiều không gian thấp hơn trong Tam Thể" |
| 复利 (lãi kép) | Dùng thẳng "lãi kép" — người Việt hiểu |
| 一句话总结 | "Tóm lại trong một câu" |
| 玄学 (huyền học) | "huyền học" hoặc "bí ẩn khó giải thích" |

### Tham chiếu văn hoá

- **Tam Thể (三体)**: Người Việt biết → giữ nguyên, thêm giải thích ngắn nếu cần
- **Harry Potter**: Toàn cầu biết → giữ nguyên
- **IKEA**: Toàn cầu biết → giữ nguyên
- **Charlie Munger**: Có thể thêm "(nhà đầu tư huyền thoại)" lần đầu nhắc đến

### Cách diễn đạt

| Gốc (Trung) | Dịch tự nhiên (Việt) |
|-------------|---------------------|
| 说实话状态很真实 | Nói thật, thực trạng rất "thật" |
| 脑子里装不下了 | Đầu óc bạn không còn chứa nổi nữa |
| 时间被切得很碎 | Thời gian bị xắt vụn |
| 松一口气 | Thở phào nhẹ nhõm |
| 这件事本质上是低效的 | Bản chất của chuyện này là sự lãng phí |

## Cấu trúc bài dịch

1. **Giữ nguyên heading hierarchy** — mỗi `##` / `###` trong gốc phải có tương ứng trong bản dịch
2. **Giữ nguyên số đoạn văn** — không gộp hoặc tách đoạn tuỳ tiện
3. **Giữ nguyên vị trí `[[IMG_N]]`** — marker phải nằm đúng vị trí tương đối so với text xung quanh
4. **Có thể thêm**: subheading phụ nếu đoạn quá dài, callout cho insight quan trọng

## Đánh dấu format khi dịch

Trong quá trình dịch, đánh dấu nội dung phù hợp cho Lark formatting:

- **Insight quan trọng** → sẽ thành `<callout emoji="💡" background-color="light-blue">`
- **Cảnh báo / lưu ý** → sẽ thành `<callout emoji="⚠️" background-color="light-yellow">`
- **Kết luận mạnh** → sẽ thành `<callout emoji="✅" background-color="light-green">`
- **Trích dẫn gốc** → sẽ thành `<callout emoji="📌" background-color="light-purple">`
- **So sánh song song** → sẽ thành `<grid cols="2">`
- **Danh sách có thuộc tính** → sẽ thành bảng Markdown
