# 6. Ngân hàng câu hỏi phỏng vấn & trả lời mẫu

70+ câu, chia nhóm, từ dễ đến khó. Mỗi câu có gợi ý trả lời. Đọc rồi tự diễn đạt lại.

> ⭐ = câu rất hay được hỏi.

---

## A. Tổng quan & động lực

**A1. ⭐ Giới thiệu ngắn gọn về dự án.**
> Quán quen là web app PWA lưu quán ăn/cafe theo gu cá nhân trên bản đồ — chấm điểm theo tiêu chí riêng (ngon/yên tĩnh/ổ cắm/date/làm việc), gợi ý "hôm nay đi đâu", lập lịch trình với tuyến đường thật, khám phá quán mới từ OpenStreetMap. Vanilla JS + Leaflet, không backend, dữ liệu ở localStorage + IndexedDB, backup mã hóa, đồng bộ Gist tùy chọn. Có 47 test Playwright + CI/CD.

**A2. Vì sao làm dự án này?**
> Google Maps không nhớ "gu" cá nhân. Em muốn một sổ tay quán riêng tư theo cảm nhận của mình, và đây cũng là cơ hội thực hành bản đồ tương tác, lưu trữ nâng cao, tích hợp nhiều dịch vụ ngoài và kỷ luật kỹ thuật (test + CI).

**A3. Đối tượng người dùng?**
> Người hay đi cafe/quán ăn, muốn nơi riêng tư ghi lại quán theo cảm nhận cá nhân — không tài khoản, không server.

**A4. ⭐ Khác gì Google Maps?**
> Google Maps chấm điểm chung của cộng đồng; Quán quen chấm theo **gu cá nhân** đa tiêu chí, gợi ý theo mood, dữ liệu **riêng tư** ở máy bạn, và bạn kiểm soát hoàn toàn.

---

## B. Kiến trúc & thiết kế

**B1. ⭐ Mô tả kiến trúc tổng thể.**
> Client-only, không backend. index.html chứa DOM tĩnh; i18n.js từ điển dịch; app.js là bộ não (state + init + render + logic); style.css theme; sw.js offline. Bản đồ bằng Leaflet. Dữ liệu ở localStorage (text) + IndexedDB (ảnh). Gọi Nominatim/Overpass/OSRM/Gist khi cần.

**B2. ⭐ Vì sao không backend?**
> Không cần — app cá nhân, dữ liệu riêng tư nên giữ ở máy là hợp lý nhất. Lợi: riêng tư tuyệt đối, deploy đơn giản (tĩnh trên GitHub Pages), không server bảo trì. Đồng bộ đa thiết bị giải bằng GitHub Gist tùy chọn.

**B3. ⭐ Vì sao vanilla JS, không framework/bundler?**
> App phức tạp về tính năng nhưng state quản được bằng mô hình "render() đọc state vẽ UI". Không bundler → deploy = file tĩnh, và test Playwright gọi thẳng hàm global qua page.evaluate. Đổi lại tự quản DOM/state — em bù bằng 47 test tự động.

**B4. Mô hình render hoạt động thế nào?**
> Mọi thay đổi state gọi render(); render() tính getFilteredPlaces() rồi vẽ lại list, markers, heat, route, gợi ý, thống kê. "UI là hàm của state" như React nhưng thủ công. Marker được tái sử dụng (chỉ thêm/xóa/cập nhật phần khác) để khỏi vẽ lại từ đầu.

**B5. app.js 4300 dòng — nhược điểm? Khắc phục?**
> File lớn khó bảo trì. Hướng cải thiện: tách ES modules + bundler. Hiện giữ một file để không cần build và để test gọi hàm global dễ. Em ghi nhận đây là giới hạn.

---

## C. Lưu trữ (localStorage + IndexedDB)

**C1. ⭐ Vì sao dùng cả localStorage và IndexedDB?**
> localStorage chỉ ~5MB và lưu text; ảnh base64 rất nặng sẽ làm đầy ngay. Nên ảnh để IndexedDB (dung lượng lớn, async), còn dữ liệu quán text ở localStorage. Trong localStorage trường images chỉ giữ {id, category}, không giữ dataUrl.

**C2. ⭐ Luồng ảnh hoạt động ra sao?**
> Khởi động: loadImagesFromDb() nạp ảnh vào imageCache (Map RAM), hydratePlaceImages() gắn dataUrl vào từng quán. Khi lưu: savePlaces() strip dataUrl khỏi localStorage + persistImagesToDb() ghi ảnh mới vào IndexedDB. pruneOrphanImages() dọn ảnh không còn quán tham chiếu.

**C3. IndexedDB khác localStorage thế nào?**
> IndexedDB: async, chứa object/blob, dung lượng lớn; localStorage: đồng bộ, chỉ text, ~5MB. IndexedDB phức tạp hơn (open/transaction/store) nhưng hợp dữ liệu lớn như ảnh.

**C4. Có rủi ro gì khi xóa dữ liệu trình duyệt?**
> Mất hết (vì không server). Nên app có backup JSON/CSV/mã hóa + đồng bộ Gist + nhắc backup định kỳ (mỗi 7 ngày).

---

## D. Bản đồ (Leaflet)

**D1. ⭐ Leaflet là gì? Dùng thế nào?**
> Thư viện bản đồ tương tác mã nguồn mở. Em dùng tileLayer (nền OSM), marker cho quán (icon + popup tùy biến), markercluster gom cụm khi nhiều quán gần nhau, leaflet.heat cho heatmap mật độ.

**D2. Tile bản đồ là gì? Cache thế nào?**
> Bản đồ chia lưới ô ảnh 256px theo zoom (z/x/y); chỉ tải ô đang xem. SW cache tile đã xem (cache-first) giới hạn 240 ô (trimCache) để xem lại offline mà không phình cache vô hạn.

**D3. Heatmap và markercluster giải quyết vấn đề gì?**
> markercluster: tránh rối khi quá nhiều marker chồng nhau (gom thành cụm số, bấm tách). Heatmap: nhìn nhanh mật độ quán theo vùng màu thay vì từng marker.

**D4. Vì sao gọi map.invalidateSize() khi thu/mở sidebar?**
> Leaflet cần biết container đổi kích thước để vẽ lại tile đúng; nếu không bản đồ bị méo/xám một phần.

---

## E. Dịch vụ ngoài & mạng

**E1. ⭐ Kể các dịch vụ ngoài và vai trò.**
> OSM tiles (nền), Nominatim (địa chỉ↔toạ độ), Overpass (tìm quán thật), OSRM (định tuyến đường phố), GitHub Gist (đồng bộ tùy chọn).

**E2. ⭐ fetchWithRetry làm gì?**
> Gọi fetch, nếu 429/503 thì đợi (900ms×lần) rồi thử lại; hết lượt ném lỗi có cờ rateLimited để UI báo "bị giới hạn tần suất". Tăng độ bền khi phụ thuộc dịch vụ dùng chung.

**E3. Nếu OSRM/Overpass lỗi hoặc offline thì sao?**
> Có fallback nhiều tầng: OSRM lỗi → giữ đường chim bay; "Random gần tôi" lỗi → gợi ý từ danh sách đã lưu; offline → vẫn dùng quán đã lưu (chỉ không tìm quán mới được).

**E4. Vì sao SW fetch thẳng Nominatim/Overpass/OSRM mà không cache?**
> Dữ liệu động, theo thời gian thực và có điều khoản dịch vụ; cache sẽ trả kết quả cũ. Chỉ cache tile (ít đổi).

**E5. Khám phá quán hoạt động thế nào?**
> discoverArea gọi Overpass với truy vấn `node["amenity"~"cafe|restaurant|..."]["name"](around:radius,lat,lng)`, parseOverpassPois lọc POI hợp lệ, guessTypeFromTags đoán loại, poiToDraft điền sẵn form để lưu nhanh.

---

## F. Bảo mật & riêng tư

**F1. ⭐ Backup mã hóa hoạt động ra sao?**
> encryptBackup dùng PBKDF2-SHA-256 250.000 vòng (dẫn khóa từ mật khẩu + salt ngẫu nhiên) rồi AES-GCM (iv ngẫu nhiên). File lưu salt/iv/data base64. Khôi phục cần đúng mật khẩu.

**F2. Vì sao PBKDF2 250k vòng?**
> Làm chậm việc dò mật khẩu (brute force): mỗi lần thử tốn 250k vòng băm. Salt chống bảng tra cứu sẵn (rainbow table).

**F3. Dữ liệu người dùng có an toàn không?**
> Mọi dữ liệu ở máy (localStorage/IndexedDB), không gửi đi đâu trừ khi bật Gist. Token GitHub lưu cục bộ, chỉ gửi tới api.github.com.

**F4. Chống XSS thế nào?**
> Dữ liệu động (kể cả từ Overpass/Nominatim) được escapeHtml/escapeAttr trước khi render, và chuẩn hóa qua normalizePlace.

**F5. Vì sao không dùng prompt()/confirm() native?**
> Để UX nhất quán và test được. App tự viết showPrompt/showConfirm dạng modal; thậm chí có test kiểm tra "tạo bộ sưu tập không bật dialog native".

---

## G. Testing & CI/CD (trọng tâm của dự án)

**G1. ⭐ Dự án test thế nào?**
> 47 test Playwright (E2E) điều khiển trình duyệt thật, kiểm tra render/import/backup/undo/theme/i18n/lọc/lịch trình/OSRM/CSV/IndexedDB/Gist... Test mạng mock window.fetch. Thêm check-encoding.js chống mojibake và node --check cú pháp.

**G2. ⭐ Vì sao test gọi được hàm trong app trực tiếp?**
> Vì app.js là classic script, mọi hàm ở global scope; Playwright dùng page.evaluate chạy code trong trang nên gọi thẳng được (vd createBackupPayload, normalizePlace, getMoodScore).

**G3. Làm sao test phần gọi mạng mà không phụ thuộc mạng?**
> Mock: gán window.fetch = async () => ({ ok:true, json: async()=>(...) }) trong test, kiểm tra app xử lý đúng. Ví dụ test gist push/pull, test fetchWithRetry trả 429.

**G4. ⭐ CI/CD của bạn ra sao?**
> GitHub Actions: push lên main → chạy npm run check + npm test; chỉ khi pass thì job deploy-pages mới deploy GitHub Pages. Nên không bao giờ deploy bản lỗi JS/manifest/encoding/test.

**G5. Một test thú vị?**
> "images are stored in indexeddb and stripped from localStorage": tạo quán có ảnh, savePlaces + persistImagesToDb, rồi kiểm tra localStorage KHÔNG chứa "data:image", còn imageCache thì có → chứng minh kiến trúc 2 tầng đúng.

**G6. Vì sao test "lucide phải ghim version"?**
> Phòng ngừa rủi ro chuỗi cung ứng/đổi API: nếu dùng @latest, thư viện đổi có thể phá app âm thầm. Test đảm bảo URL có version cố định, không @latest.

---

## H. PWA & i18n

**H1. PWA của bạn cache gì?**
> App shell (html/css/js/icon) + thư viện CDN (Leaflet/markercluster/heat/lucide) + tile đã xem. 4 chiến lược theo origin (xem câu E4).

**H2. share_target là gì?**
> Khai báo trong manifest để khi cài như PWA, app nhận link được chia sẻ từ điện thoại (vd chia sẻ địa điểm Google Maps) → tự import.

**H3. i18n hoạt động thế nào?**
> I18N_STRINGS {vi,en} trong i18n.js; t(key,params) trả chuỗi theo lang. UI tĩnh dùng data-i18n + applyTranslations(); chuỗi động gọi t() trực tiếp. toggleLang lưu localStorage + dịch lại + render.

**H4. Người dùng nhận bản cập nhật thế nào?**
> Bump CACHE_VERSION; SW mới dọn cache cũ; nút "Cập nhật" hiện khi có SW chờ, bấm gửi message SKIP_WAITING để kích hoạt ngay.

---

## I. Câu hỏi mở / tư duy

**I1. ⭐ Thử thách khó nhất?**
> (Gợi ý) Quản lý ảnh hai tầng (localStorage giới hạn → IndexedDB + hydrate + dọn mồ côi) cho mượt; hoặc làm 47 test E2E ổn định với phần phụ thuộc mạng (phải mock fetch). Kể vấn đề → cách giải → kết quả.

**I2. ⭐ Có thêm thời gian sẽ cải thiện gì?**
> Tách app.js thành ES modules + bundler; thêm test accessibility với screen reader; tối ưu Overpass (cache theo ô lưới); có thể thêm backend tùy chọn cho đồng bộ realtime đa thiết bị thay vì Gist thủ công.

**I3. App "vỡ" ở đâu khi dữ liệu lớn / mạng kém?**
> localStorage ~5MB cho text vẫn có giới hạn (có cảnh báo khi gần đầy); Overpass/OSRM có thể chậm/giới hạn (đã có retry + fallback). Quá nhiều marker thì markercluster + heatmap đỡ rối.

**I4. Quyết định kỹ thuật tự hào nhất?**
> Kỷ luật kỹ thuật: 47 test Playwright + CI chặn deploy bản hỏng + kiểm encoding. Nó cho thấy em xây sản phẩm bền chứ không chỉ "chạy được".

**I5. Trade-off lớn nhất?**
> No-backend (riêng tư, đơn giản) đổi lại đồng bộ đa thiết bị phải thủ công (Gist). No-bundler (deploy dễ, test gọi hàm global) đổi lại app.js lớn khó mở rộng.

---

## J. Câu hỏi "code cụ thể"

**J1. getFitScore vs getMoodScore khác gì?**
> getFitScore: điểm "hợp gu" chung (trung bình rating theo gu đang lọc + boost ruột/mới ghé) — hiện trên marker/list. getMoodScore: điểm theo mood gợi ý cụ thể (work/date/quiet/nearby/new...) — dùng cho "hôm nay đi đâu".

**J2. Reroll gợi ý có thật ngẫu nhiên?**
> Không — random có trọng số: điểm cao → trọng số cao (score^1.25), quán vừa ghé trong 14 ngày bị phạt ×0.22 → đa dạng nhưng vẫn hợp lý.

**J3. normalizePlace để làm gì?**
> Chuẩn hóa mọi quán từ mọi nguồn (nhập tay/import/OSM/backup): điền mặc định, ép kiểu, lọc rating 0-5, tag tối đa 12, validate ngày... → dữ liệu trong app luôn đồng nhất, an toàn.

**J4. parseCoordsString xử lý gì?**
> Tách chuỗi "lat, lng" (copy từ Google Maps), bỏ ngoặc/`;`/`|`, regex bắt 2 số, kiểm tra lat∈[-90,90], lng∈[-180,180]; sai trả null.

**J5. getItineraryDirectionsUrl dựng URL thế nào?**
> Dựng URL Google Maps dir đa điểm: origin = vị trí hiện tại (nếu có) hoặc điểm đầu, destination = điểm cuối, waypoints = các điểm giữa nối bằng `|`.

**J6. savePlaces() làm gì đặc biệt?**
> Strip ảnh: serialize quán với images chỉ {id, category} (bỏ dataUrl) trước khi ghi localStorage, rồi persistImagesToDb() ghi ảnh vào IndexedDB, và renderDataPanel().

---

## 🎤 5 câu hay gặp nhất — luyện trước tiên
1. A1 — Giới thiệu dự án.
2. C1/C2 — Vì sao localStorage + IndexedDB & luồng ảnh.
3. G1/G4 — Test Playwright & CI/CD.
4. F1 — Backup mã hóa.
5. E2/E3 — fetchWithRetry & fallback mạng.

## ❓ Câu hỏi NÊN HỎI LẠI nhà tuyển dụng
- Team có pipeline CI/CD và mức độ phủ test thế nào?
- Sản phẩm dùng bản đồ/geospatial không, stack ra sao (Leaflet/Mapbox/Google)?
- Cách team xử lý lưu trữ phía client và đồng bộ offline?
