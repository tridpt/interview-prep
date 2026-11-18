# 06 — Câu hỏi phỏng vấn

**1. Có thực sự jailbreak LLM không?** Mặc định không. `engine.js`/`indirect.js` là simulator regex/điểm/ngưỡng. Sandbox có thể gọi LLM thật theo lựa chọn người dùng, nhưng 12 màn không đo tỷ lệ thành công của model thật.

**2. Direct khác indirect thế nào?** Direct gửi câu người tấn công trong kênh chat; indirect chèn chỉ thị vào tài liệu được truy xuất, nơi ranh giới dữ liệu/chỉ thị dễ nhầm. `runAttack` và `runIndirect` là hai engine riêng.

**3. Vì sao output scan chưa đủ?** Trong simulator, scanner so chuỗi bí mật nguyên văn; mã hóa hoặc biến đổi có thể qua. Biện pháp căn bản là không để bí mật nằm trong phạm vi dữ liệu/tool mà nội dung ngoài có thể kích hoạt.

**4. Provenance là gì?** Đánh dấu nguồn và quyền hạn của nội dung; tài liệu truy xuất chỉ là dữ liệu. `runIndirect` mô phỏng chặn một số pattern nhưng không bảo đảm an toàn tuyệt đối.

**5. Multi-turn lưu trạng thái ra sao?** `runAttack` trả `state.primed`; UI mang sang lượt tiếp theo. Đừng nhầm với session server hoặc model có memory dài hạn.

**6. Key của LLM đi đâu?** `llm.js` gọi provider từ client khi bật model thật. Key ở bộ nhớ trình duyệt trong phiên; không nhập secret thật, cân nhắc rủi ro gọi API bên thứ ba.

**7. Test gì?** `npm test` chạy `engine.test.mjs`, `indirect.test.mjs`, `store.test.mjs` bằng Node; chưa phải E2E trình duyệt hay kiểm toán LLM thực tế.
