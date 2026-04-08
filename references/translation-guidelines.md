# Hướng dẫn dịch thuần Việt

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
