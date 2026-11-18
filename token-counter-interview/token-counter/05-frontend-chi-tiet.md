# 05 — Chi tiết Frontend (đọc từng phần)

Frontend là HTML/CSS/JS thuần, không framework. Gồm `index.html`, `app.js`, `style.css`.

---

## A. `index.html` — bố cục

Giao diện chia thành các khối:
1. **Topbar**: tiêu đề + nút "Scan Local Logs" + ô trạng thái (`#status`).
2. **Metric grid** (4 ô lớn): Total tokens, Input tokens, Output tokens, Kiro credits.
3. **Panel "Sources"**: danh sách nguồn đã quét (Claude/Kiro/Antigravity) kèm badge trạng thái.
4. **Panel "Recent usage records"**: bảng các bản ghi (Time, Source, Model, Input, Output, Total, Credits, Context).
5. **Panel "Độ chính xác"**: ghi chú trung thực về giới hạn dữ liệu.
6. **Panel "Paste API usage JSON"**: ô dán JSON + nút Parse + 3 ô kết quả (Input/Output/Total).
7. **Panel "Manual tokenizer estimate"**: chọn model, 2 ô Input/Output, các con số token + encoding.

> Lưu ý: phần tĩnh viết bằng tiếng Việt không dấu (vd "Doc usage thuc te...") — chi tiết đặc thù của dự án, không ảnh hưởng logic.

## B. `app.js` — logic chính

### Lấy tham chiếu DOM + tiện ích
- Gom mọi `getElementById` lên đầu.
- `formatNumber`: dùng `Intl.NumberFormat("en-US")` để hiển thị số có dấu phẩy ngăn cách.
- `formatTime`: `new Date(value).toLocaleString()`, fallback nếu không parse được.
- `escapeHtml`: **chống XSS** — thay `& < > " '` thành entity trước khi nhét vào `innerHTML`. (Điểm bảo mật đáng nói: dữ liệu từ log là không tin cậy.)

### `api(path, options)` — bọc fetch
```js
const res = await fetch(API + path, { headers: {"Content-Type":"application/json"}, ...options });
const data = await res.json().catch(() => ({}));
if (!res.ok) throw new Error(data.detail || "Request failed");
return data;
```
Tập trung xử lý lỗi và header ở một chỗ.

### `scanUsage()` + `renderScan(data)`
- `scanUsage`: đặt trạng thái "Scanning", gọi `/api/usage/scan`, render, đặt lại trạng thái; bắt lỗi hiển thị.
- `renderScan`: 
  - Cập nhật 4 metric tổng từ `data.totals`.
  - Dựng từng **source card** (tên, badge `status`, note, đường dẫn, số file/records).
  - Dựng **bảng records** (tối đa 250 dòng để nhẹ DOM), mỗi dòng `title = record.path` để hover xem nguồn.
  - Ẩn/hiện thông báo "No usage records" theo số bản ghi.

### `loadModels()` + `countTokens()`
- `loadModels`: gọi `/api/models`, đổ vào `<select>`, mặc định `gpt-4o-mini`.
- `countTokens`: cập nhật số ký tự ngay, gọi `/api/count`, hiển thị input/output/total token + encoding. Lỗi thì hiện message ở chỗ encoding.

### `scheduleCount()` — debounce
```js
clearTimeout(debounceTimer);
debounceTimer = setTimeout(countTokens, 220);
```
Mỗi lần gõ, hủy hẹn cũ và đặt hẹn mới 220ms → chỉ gọi API khi ngừng gõ. Tránh spam request.

### Parse JSON usage (chạy ở **client**)
Đây là bản sao logic backend nhưng bằng JS, để parse nhanh ngay trên trình duyệt:
- `looksLikeUsage(value)`: dict có khóa usage nào không.
- `findUsage(value)`: **đệ quy** tìm object usage (ưu tiên `value.usage`, `value.usageMetadata`, rồi duyệt con).
- `pickNumber(obj, keys)`: lấy số đầu tiên hợp lệ theo danh sách khóa.
- `parseUsageJson()`: parse text, tìm usage, rút input/output/total (total = lấy trực tiếp hoặc cộng input+output), hiển thị; lỗi thì hiện `usageError`.

> Điểm hay: logic dò usage được **nhân đôi** ở cả backend (Python) và frontend (JS) với cùng triết lý "thử nhiều tên khóa + đệ quy". Khi phỏng vấn có thể nói về đánh đổi này (tiện cho client nhưng phải đồng bộ 2 nơi).

### Gắn sự kiện + khởi động
```js
loadModels()
  .then(() => Promise.all([countTokens(), scanUsage()]))
  .catch(...);
```
- Nút Scan/Count/Clear/Parse, đổi model, gõ input/output đều có listener.
- Ô dán JSON cũng debounce 220ms.
- Khi tải trang: nạp model → đồng thời đếm (rỗng) và quét log.

## C. `style.css`
- Theme sáng (light), biến CSS (`--bg`, `--panel`, `--blue`...), bóng đổ mềm.
- Lưới metric 4 cột, bảng records cuộn ngang khi hẹp.
- Badge trạng thái đổi màu theo `ok` (xanh) / `no_usage` (vàng) / `missing` (đỏ).
- Responsive: 2 cột rồi 1 cột khi màn hình nhỏ.

## D. Điểm đáng nói khi phỏng vấn về frontend
1. **`escapeHtml` chống XSS**: dữ liệu log không tin cậy, nên thoát HTML trước khi render — tư duy bảo mật tốt.
2. **Debounce**: trải nghiệm mượt, không spam backend khi gõ.
3. **Giới hạn render 250 dòng**: giữ DOM nhẹ dù có hàng nghìn bản ghi.
4. **Tách trạng thái rõ ràng**: ô `#status` đổi `data-kind` (ok/error) để CSS tô màu.
5. **Không framework**: chứng tỏ nắm DOM API thuần, fetch, async/await.
