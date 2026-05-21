# Anti-pattern: phát hiện và sửa dịch sát chữ

> **Mục đích:** Bảng tra cứu cụ thể những cách dịch SAI thường gặp khi dịch Trung → Việt. Mỗi lần dịch xong PHẢI scan bản dịch để tìm các pattern này và rewrite.
>
> Đây không phải khuyến nghị — đây là **danh sách bắt buộc kiểm tra** trước khi `lark-cli docs +create`.

---

## 1. Calques từ tiếng Trung (cấu trúc câu Trung)

| ❌ Sai (calque) | ✅ Đúng (tự nhiên) | Gốc | Lý do |
|----|----|----|----|
| Một là bản flagship: ... Một là bản nhẹ: ... | **Phiên bản flagship:** ... **Phiên bản nhẹ:** ... | 一个是旗舰版 ... 一个是轻量版 | "Một là... Một là..." là kết cấu liệt kê tiếng Trung, không dùng trong tiếng Việt |
| Trước hết nói về X | Hãy nói về X trước / Bắt đầu với X | 先说 X | "Trước hết" + verb là cách nói báo cáo, không tự nhiên trong văn nói |
| Việc nào dùng được bản nhẹ thì dùng bản nhẹ | Dùng được phiên bản nhẹ thì dùng | 能用轻量版就用轻量版 | "Việc nào... thì..." nặng nề |
| đi theo cùng một công thức | đi theo cùng một công thức / đều giống một kiểu | 同一个套路 | "công thức" OK, nhưng có thể dùng "chiêu" / "kiểu" tự nhiên hơn |
| Bài toán thực sự cần flagship | Bài toán bắt buộc cần flagship / Việc thực sự khó phải dùng flagship | 非要用旗舰版的难题 | Trật tự "thực sự cần X" không phải tiếng Việt tự nhiên |
| Cắn răng cũng phải dùng | Bấm bụng dùng / Cắn răng chi tiền dùng | 咬咬牙也得用 | "Cắn răng" đứng một mình thiếu cụm, cần kết hợp |

---

## 2. Động từ dịch sát chữ

| ❌ Sai | ✅ Đúng | Gốc | Lý do |
|----|----|----|----|
| **vượt ngược** Gemini 3.1 Pro | **vượt mặt** Gemini 3.1 Pro / **vượt qua** | 反超 | "Vượt ngược" không có trong tiếng Việt. "反超" = vượt mặt đối thủ |
| nhìn thấy Flash sẽ nghĩ ngay đến | nghe thấy / nghe nhắc đến / nói đến Flash là nghĩ ngay đến | 一看到 Flash | "看到" dịch máy = "nhìn thấy", nhưng văn cảnh ở đây là "khi nhắc đến" |
| đưa cái này lên / khiêng lên video | đưa cách làm đó lên video / áp dụng vào video | 把这一套搬到了视频上 | "搬" = di chuyển, không phải "khiêng" |
| thực hiện tăng trưởng | tăng trưởng / đạt mức tăng | 实现增长 | "Thực hiện" + verb là calque |
| rơi xuống đất / triển khai mặt đất | triển khai thực tế / áp dụng vào thực tế | 落地 | "落地" idiom = từ ý tưởng thành sản phẩm thật |
| đứng trên cao đánh chiều thấp | đánh kiểu chiều không gian thấp hơn (như Tam Thể) | 降维打击 | Giải thích bằng tham chiếu Tam Thể (người Việt biết) |
| nhồi nhét / ấn vào | đưa vào / nén vào / nhét vào | 装进 | Tùy ngữ cảnh — "装进速度和价格里" = "nhét vào tốc độ và giá" |

---

## 3. Cụm chuyển ý / liên kết câu

| ❌ Sai | ✅ Đúng | Gốc |
|----|----|----|
| Đối với mỗi cá nhân mà nói | Với từng người / Với mỗi người | 对每个人而言 |
| Về phần X | Phía X / Còn X / X thì | X 这边 |
| Cũng tức là nói | Tức là / Nói cách khác | 也就是说 |
| Trên thực tế | Thực ra / Trên thực tế (OK nếu trang trọng) | 其实 / 实际上 |
| Là vì sao? | Vì sao? / Tại sao thế? | 是为什么? |
| Sở dĩ X là vì Y (lạm dụng) | X là vì Y / Lý do X là Y | 之所以 X 是因为 Y |
| Nói thẳng ra là | Nói thẳng / Thẳng thắn mà nói | 说白了 |
| Một mặt... mặt khác... | Một mặt... mặt khác... (OK nhưng dùng ít thôi) | 一方面...另一方面... |

---

## 4. Danh từ — chọn từ Hán Việt hay thuần Việt

Quy tắc: **có từ thuần Việt thì dùng thuần Việt**. Hán Việt chỉ giữ khi không thay được hoặc đã chuẩn hoá ngành.

| ❌ Hán Việt kiêu | ✅ Tự nhiên | Khi nào giữ Hán Việt |
|----|----|----|
| đại đa số người dùng | hầu hết người dùng / phần lớn | "đại đa số" OK trong văn nghị luận chính trị |
| toàn cầu trên thế giới | toàn cầu / cả thế giới | "toàn cầu" được, đừng kèm "trên thế giới" (thừa) |
| phương thức sử dụng | cách dùng / cách sử dụng | "phương thức" OK trong tài liệu kỹ thuật |
| phát huy tác dụng | có tác dụng / phát huy hiệu quả | "phát huy" OK với "vai trò", "thế mạnh" |
| tiến hành phân tích | phân tích | Bỏ "tiến hành" — verb đứng một mình đủ |
| khả năng tiến hành... | có thể... | Bỏ "tiến hành" |
| thực hiện việc... | làm... / xử lý... | Bỏ "thực hiện việc" |
| đảm bảo rằng X | đảm bảo X / chắc chắn X | "rằng" thường thừa |
| trong việc... | khi... / trong (verb)... | "trong việc X-ing" thường có thể bỏ "việc" |
| khối lượng lớn / hàng khối lượng lớn | số lượng lớn / hàng loạt / nhiều | 大批量 |
| nhiệm vụ phức tạp | tác vụ phức tạp / công việc phức tạp | Tùy ngữ cảnh kỹ thuật |

---

## 5. Văn phong câu (cấu trúc)

### 5a. Bị động lạm dụng

❌ `Tính năng này được phát triển bởi Google.`
✅ `Google phát triển tính năng này.` / `Tính năng này do Google phát triển.`

Tiếng Trung dùng `被` ít, tiếng Anh dùng passive nhiều — tiếng Việt nên giảm passive, ưa chủ động.

### 5b. Câu quá dài, nhiều mệnh đề lồng

❌ Câu Trung 60-80 ký tự thường có 3-4 mệnh đề lồng. Dịch nguyên là **câu Việt 80+ ký tự không có dấu chấm** = mệt đọc.

✅ Tách thành 2-3 câu ngắn. Mỗi câu một ý. Đọc to lên phải nghe trôi.

**Ví dụ:**
❌ `Nó là mô hình agent và lập trình mạnh nhất hiện tại của Google đạt 76.2% trên Terminal Bench 2.1 và GDPval AA 1656 Elo trên MCP Atlas 83.6% và CharXiv Reasoning 84.2%, đồng thời tốc độ tạo token nhanh gấp 4 lần các mô hình frontier khác với giá khoảng bằng nửa.`

✅ `Google gọi Gemini 3.5 Flash là mô hình agent và lập trình mạnh nhất của họ hiện tại: Terminal Bench 2.1 đạt 76.2%, GDPval AA 1656 Elo, MCP Atlas 83.6%, CharXiv Reasoning 84.2%.`
`Về tốc độ, Flash tạo ra khoảng 289 token/giây — nhanh gấp 4 lần các mô hình frontier khác.`
`Giá khoảng bằng nửa các mô hình frontier khác.`

### 5c. Trật tự bổ ngữ

❌ `Ở Google Antigravity, ở 3.5 Flash, có thể triển khai...` — trật tự dồn dập như Trung
✅ `Đặc biệt trong Google Antigravity, 3.5 Flash có thể triển khai...`

### 5d. Đại từ chỉ định không cần thiết

❌ `Cái sự thay đổi chi phí này, nhìn dài hạn, có ảnh hưởng cái này lớn hơn...`
✅ `Sự thay đổi chi phí này, nhìn dài hạn, ảnh hưởng lớn hơn...`

Tiếng Trung dùng `这`, `这个`, `那` nhiều — tiếng Việt bỏ được thì bỏ.

---

## 6. Glossary AI/Tech (zh → vi)

Term dùng nhất quán xuyên suốt:

| Tiếng Trung | Tiếng Việt chuẩn |
|----|----|
| 模型 | mô hình |
| 大模型 | mô hình lớn / LLM |
| 跑分 | điểm chuẩn / benchmark |
| 跑分超过 | điểm vượt qua / điểm chuẩn vượt mặt |
| 上下文 | context |
| 推理 | suy luận / inference (tùy ngữ cảnh) |
| 智能体 | agent (giữ English) |
| 子智能体 | sub-agent / agent con |
| 工作流 | workflow |
| 提示词 | prompt |
| 多模态 | đa phương thức / multimodal |
| 微调 | fine-tune / tinh chỉnh |
| 蒸馏 | distill / chưng cất (mô hình) |
| 量化 | quantize / lượng tử hoá |
| 训练 | train / huấn luyện |
| 部署 | deploy / triển khai |
| 算力 | compute power / sức tính toán |
| 显卡 | GPU / card đồ hoạ |
| 接入 | tích hợp / kết nối |
| 调用 | gọi (API/tool) |
| 落地 | triển khai thực tế / áp dụng |
| 上线 | ra mắt / khả dụng / lên live |
| 内测 | beta nội bộ / thử nghiệm hẹp |
| 公测 | public beta / công khai beta |
| 灰度 | rollout dần / triển khai tăng dần |
| 默认 | mặc định |
| 用户 | người dùng |
| 客户 | khách hàng |
| 套路 | công thức / chiêu / kiểu |
| 入口 | cửa ngõ / entry point |
| 全家桶 | trọn bộ / cả gia đình |
| 玩法 | cách dùng / cách chơi |
| 体验 | trải nghiệm |

**Lưu ý:** Tên sản phẩm/công ty/ngành giữ tiếng Anh: Gemini, Spark, Omni, Antigravity, AI Studio, MCP, OCR, VFX, API, GPU, LLM, RAG, etc.

---

## 7. Idiom Trung — cách giải thích cho người Việt

| Idiom | Nghĩa | Cách dịch |
|----|----|----|
| 降维打击 | đánh chiều không gian thấp hơn | "Đánh kiểu chiều không gian thấp hơn (như trong Tam Thể)" |
| 弯道超车 | vượt xe ở khúc cua | "Vượt mặt ở khúc cua" / "Đi đường tắt vượt lên" |
| 一招鲜 | một chiêu tinh | "Một mánh duy nhất" / "Một thế mạnh độc nhất" |
| 三板斧 | ba búa rìu (Cheng Yaojin) | "Ba đòn cơ bản" |
| 卷 / 内卷 | involution | "Cạnh tranh nội bộ căng thẳng" / "đua nhau X" |
| 躺平 | nằm phẳng | "Buông xuôi" / "Bỏ cuộc theo cách thư giãn" |
| 上头 | lên đầu (nghiện) | "Cuốn vào" / "Mê man" |
| 破圈 | phá khỏi vòng tròn | "Tiếp cận người ngoài ngành" / "Vượt khỏi cộng đồng nhỏ" |
| 全村人的希望 | hy vọng của cả thôn | "Hy vọng của cả ngành" / "Niềm hy vọng duy nhất" |
| 杀手锏 | tuyệt chiêu (Sát Thủ Giản) | "Át chủ bài" / "Tuyệt chiêu" |

---

## 8. Quy trình bắt buộc trong Phase 3

### Pass 1 — Dịch nghĩa
Dịch nội dung sát ý, giữ các `**bold**` và `[[IMG_N]]` markers.

### Pass 2 — Tự rewrite (BẮT BUỘC)

Đọc lại từng đoạn vừa dịch, hỏi 5 câu:

1. **Có cụm nào trong bảng anti-pattern (mục 1-5) không?** → Sửa ngay
2. **Đọc to câu này lên có nghe trôi không?** → Nếu vấp, rewrite
3. **Có ai người Việt thực tế viết câu này tự nhiên không?** → Nếu không, rewrite
4. **Có chữ Hán Việt nào có thể thay bằng thuần Việt không?** → Có thì thay
5. **Có câu nào dài > 40 từ không?** → Có thì tách

### Pass 3 — Glossary consistency check

Scan toàn bài, đảm bảo các term ở mục 6 dùng nhất quán. Vd: nếu chọn dịch "agent" thì xuyên suốt "agent", không lúc "tác nhân" lúc "đại lý".

### Pass 4 — Format preservation (đã có)

Đếm `**...**` markers — số lượng phải khớp text gốc.

---

## 9. Test pass/fail trước khi tạo Lark doc

Sample 3 đoạn ngẫu nhiên từ bản dịch, đọc to. Nếu BẤT KỲ đoạn nào thấy:
- Có cụm trong bảng "❌ Sai" của mục 1-5
- Câu trên 40 từ không có dấu phẩy/chấm
- Có "thực hiện việc X", "tiến hành Y", "đối với mỗi cá nhân mà nói"
- Có "Một là... Một là..." liệt kê

→ **Quay lại Pass 2, rewrite. KHÔNG được nộp bài.**
