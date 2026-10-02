# Prompt: Thiết kế lại UI/UX + sửa lỗi chức năng cho MSB CreditPilot 360

Dán toàn bộ nội dung dưới đây vào Claude Code (đặt thư mục làm việc là `MSB_CreditPilot_Demo`).

---

## Vai trò & mục tiêu
Bạn là senior front-end engineer + product designer. Hãy thiết kế lại toàn bộ giao diện của `web_copilot_app.py` (app web Copilot lập Tờ trình tín dụng MB07 cho ngân hàng MSB) và sửa các lỗi chức năng liệt kê bên dưới. Người dùng là RM (cán bộ quan hệ khách hàng) làm việc hằng ngày, nên giao diện phải chuyên nghiệp, gọn, dễ quét thông tin, đúng phong cách ứng dụng ngân hàng, không "demo hackathon".

## PHẦN 1 — Sửa lỗi chức năng (làm TRƯỚC, đã kiểm tra thực tế)

Kết quả chạy thử trên Python 3.11:

1. **App không khởi động**: `web_copilot_app.py` thiếu `from typing import Any, List` (đang chạy được trên Python 3.14 nhờ annotation hoãn đánh giá). Thêm import; quét toàn file bằng `python -m pyflakes web_copilot_app.py` để tìm tên chưa định nghĩa khác.
2. **`POST /api/generate_docx` trả 500**: `NameError: name 'List' is not defined` (dòng ~706 trong `execute_generation_pipeline`). Nút "Xuất Tờ Trình MB07 (.DOCX)", tính năng cốt lõi của app, đang hỏng hoàn toàn. Sau khi sửa phải xuất được file và mở được bằng Word/python-docx.
3. **Nút "➕ Tạo Mới"**: `openNewCaseModal is not defined`. Hàm JS không tồn tại.
4. **Nút "Lưu Phần A" và "Lưu Phần B"**: `saveSectionA` / `saveSectionB is not defined`. Hàm JS không tồn tại.
5. **`POST /api/switch_case`** trả 404 "Case not found" khi body rỗng. Kiểm tra dropdown chọn hồ sơ chạy đúng với `PSD`, `GAS_SOUTH` và các hồ sơ khác trong `/api/cases`; trả 400 thay vì 404 khi thiếu tham số.
6. **`/api/narrative/generate`** lỗi 500 khi thiếu API key. Phải trả thông báo thân thiện, và UI hiển thị trạng thái "Chưa cấu hình AI" kèm hướng dẫn, không để người dùng thấy stack trace hay lỗi thô.
7. **`/api/telemetry`** hard-code `"provider": "GreenNode"` dù `.env` đang đặt `AI_PROVIDER=gemini`. Lấy provider thực tế.
8. **Dùng `alert()` cho mọi phản hồi** (Nạp Demo, Phê duyệt, Lưu ghi chú, Xuất file…). Thay toàn bộ bằng toast không chặn, có loại success/error/warning.
9. **Tailwind CDN** (`cdn.tailwindcss.com`): offline hoặc sau tường lửa thì trang hiện không CSS. Phải đóng gói CSS cục bộ (build Tailwind CLI ra file `static/app.css`, hoặc viết CSS thuần), không phụ thuộc CDN lúc chạy. Font cũng phải tự host hoặc có fallback hệ thống.
10. Chưa kiểm tra được (cần file mẫu): các luồng upload và preview PDF (`/api/preview_legal_pdf`, `confirm_legal_preview`, `confirm_financial_preview`, `confirm_cic_preview`, `preview_business_pdf`, `confirm_business_preview`, `upload_file`). Hãy test từng luồng bằng PDF trong `Tham khảo/` hoặc fixtures trong `scripts/`, báo cáo luồng nào chạy, luồng nào lỗi, rồi sửa.

Bảo mật (sửa cùng đợt): `/api/upload_file` có path traversal (`fname` chưa qua `os.path.basename`); server bind `''` (mọi interface) kèm `Access-Control-Allow-Origin: *`. Đổi sang bind `127.0.0.1`, bỏ CORS `*`, giới hạn dung lượng và đuôi file upload.

## PHẦN 2 — Thiết kế lại giao diện

### 2.1 Bố cục tổng thể (YÊU CẦU CỨNG)
- **Sidebar cố định bên trái** (rộng ~248px, thu gọn được còn ~72px chỉ còn icon). **KHÔNG dùng thanh tab ngang ở trên đầu.**
- Trong sidebar, từ trên xuống:
  1. Logo MSB + tên "CreditPilot 360" (một dòng, không để chữ bị cắt/chồng như hiện tại).
  2. Bộ chọn hồ sơ khách hàng (dropdown) + nút "Tạo hồ sơ mới".
  3. Menu điều hướng theo đúng quy trình làm việc, có icon SVG (dùng bộ Lucide hoặc Heroicons, **không dùng emoji**), mục đang chọn được tô nổi bật:
     1. Tổng quan hồ sơ
     2. Tài liệu hồ sơ (upload & bóc tách)
     3. Đối soát dữ liệu (Phần A–E)
     4. Thẩm định tín dụng AI
     5. Soạn thảo tờ trình MB07
     6. Sẵn sàng Hội đồng Tín dụng
  4. Mỗi mục có chấm trạng thái nhỏ (chưa làm / đang làm / hoàn tất) để RM thấy tiến độ cả quy trình.
  5. Cuối sidebar: trạng thái AI (provider + kết nối), nút "Nạp demo chuẩn" (chỉ hiện ở chế độ demo, kiểu nút phụ), nút thu gọn.
- **Topbar mỏng** ở khu vực nội dung (cao ~56px): breadcrumb + tên khách hàng + badge trạng thái, và **một nút hành động chính "Xuất tờ trình (.docx)"** (xuất hiện ở đây và ở màn 5, không lặp ở nhiều nơi). Nút phụ để dạng outline.
- Khu vực nội dung: max-width ~1280px, padding thống nhất, cuộn độc lập với sidebar.
- Responsive: ≥1280 sidebar mở; 768–1279 sidebar thu gọn mặc định; <768 sidebar thành drawer mở bằng nút hamburger.

### 2.2 Hệ thống thiết kế (design tokens)
- Màu thương hiệu: xanh navy đậm làm màu chủ đạo (khoảng `#0B2E6B`), một màu nhấn (đỏ MSB `#E31E24` chỉ dùng cho hành động chính và cảnh báo quan trọng, **không tô đỏ/cam tràn lan**). Nền xám rất nhạt `#F5F7FA`, card trắng, viền `#E5E9F0`.
- Màu ngữ nghĩa: success xanh lá, warning hổ phách, error đỏ, info xanh dương. Mỗi màu có nền nhạt + chữ đậm để đạt contrast WCAG AA.
- Typography: một font sans (Inter hoặc Be Vietnam Pro, hỗ trợ tiếng Việt đầy đủ), thang cỡ chữ 12/14/16/20/24/30, không dùng chữ in hoa toàn bộ trừ nhãn rất nhỏ. Số liệu tài chính dùng `font-variant-numeric: tabular-nums`, căn phải.
- Khoảng cách theo lưới 4/8px; bo góc 8px (card 12px); shadow nhẹ (1–2 mức), bỏ gradient tím/cam rực rỡ hiện tại.
- Dark mode: không bắt buộc, nhưng định nghĩa màu bằng CSS variables để thêm sau.

### 2.3 Thành phần dùng chung
Viết thành các component tái sử dụng (class CSS hoặc hàm JS render), không copy–paste:
- Button (primary / secondary / outline / ghost / danger; có trạng thái loading với spinner và disabled)
- Card, Badge/Chip trạng thái, Stat tile (KPI), Table (header cố định, hàng zebra nhẹ, số căn phải), Form field (label, helper text, lỗi inline), Modal, Toast, Empty state, Skeleton loading, Stepper tiến độ, Tooltip.
- Mọi thao tác gọi API phải có: loading state, thông báo thành công/thất bại bằng toast, chặn bấm đúp.

### 2.4 Thiết kế từng màn hình
1. **Tổng quan hồ sơ**: hàng KPI (Hạn mức đề xuất, Doanh thu, LNST, Dư nợ MSB, Xếp hạng) bằng stat tile; khối "Mức độ sẵn sàng" dạng vòng tròn/thanh tiến độ kèm danh sách việc còn thiếu (click để nhảy tới màn tương ứng); card thông tin pháp lý; bảng đề xuất cấp tín dụng & bảo đảm. Bỏ banner gradient khẩu hiệu "Bảo vệ tờ trình toàn diện" (chiếm chỗ, không có thông tin).
2. **Tài liệu hồ sơ**: 4 nhóm tài liệu (Pháp lý & ĐKKD, BCTC, CIC, Phương án kinh doanh) dạng card có vùng kéo–thả upload, trạng thái từng file (đang OCR / đã bóc tách / cần rà soát / lỗi) và nút "Xem & xác nhận". Hiển thị tiến trình bóc tách thật.
3. **Đối soát dữ liệu (A–E)**: layout hai cột: trái là danh sách trường dữ liệu theo Phần, phải là bằng chứng nguồn (tên file, trang, đoạn trích). Trường xung đột/thiếu được đánh dấu rõ, lọc nhanh "Chỉ hiện trường cần xử lý". Có "Lưu" cho từng Phần (sửa lỗi `saveSectionA/B`).
4. **Thẩm định tín dụng AI**: danh sách insight dạng card, có mức độ (quan trọng/trung bình), nguồn fact, trạng thái đã xác minh. Bảng chỉ số tài chính có sparkline hoặc biểu đồ cột 3 năm (dùng skill `dataviz` nếu có).
5. **Soạn thảo tờ trình MB07**: khối soạn thảo narrative theo từng đoạn; mỗi đoạn có nhãn trạng thái (Bản thảo AI / RM đã sửa / Đã duyệt), facts nguồn thu gọn được, chỉnh sửa inline (bỏ modal nếu không cần). Thanh hành động dính (sticky) ở dưới: "Tạo bản thảo AI", "Duyệt tất cả", "Xuất DOCX". Hiển thị checklist điều kiện trước khi xuất. Nếu AI chưa cấu hình, hiện banner hướng dẫn.
6. **Sẵn sàng Hội đồng**: các thẻ câu hỏi dự kiến dạng accordion, có ô ghi chú tự lưu (debounce) thay vì nút "Lưu ghi chú" từng thẻ; lọc theo nhóm rủi ro.

### 2.5 Nội dung & trải nghiệm
- Thống nhất thuật ngữ tiếng Việt, câu ngắn, nhãn nút bắt đầu bằng động từ. Bỏ emoji khỏi tiêu đề và nút.
- Mọi số tiền định dạng `1.234.567 triệu VND` nhất quán; ngày theo `dd/MM/yyyy`.
- Không hard-code dữ liệu demo (MST `0100000000`, SĐT `0900000000`…) trong HTML/JS; mọi giá trị lấy từ `/api/case`.
- Trạng thái rỗng và lỗi đều có hướng dẫn hành động tiếp theo.
- Hỗ trợ bàn phím: focus ring rõ, thứ tự Tab hợp lý, `aria-current` cho mục menu đang chọn, `aria-live` cho toast, thẻ `<button>`/`<nav>`/`<main>` đúng ngữ nghĩa.

### 2.6 Kiến trúc front-end
- Tách HTML/CSS/JS ra khỏi `web_copilot_app.py` thành `static/index.html`, `static/app.css`, `static/app.js` (có thể chia module theo màn hình). Server chỉ phục vụ file tĩnh và API JSON. Không đổi hợp đồng API hiện có trừ các sửa ở Phần 1.
- Không thêm framework nặng; vanilla JS (ES modules) hoặc Alpine.js/htmx là đủ. Nếu dùng thư viện, vendor vào `static/vendor/`.

## PHẦN 3 — Kiểm thử & bàn giao
1. Với **mỗi nút và mỗi liên kết** trong giao diện mới, viết kịch bản Playwright (Chromium, `--no-sandbox`) bấm thử và khẳng định: không có `pageerror`, không có request 4xx/5xx ngoài dự kiến, UI phản hồi (toast/đổi trạng thái). Lưu tại `tests/e2e/test_ui_smoke.py`.
2. Test API bằng pytest cho các endpoint đã sửa (`generate_docx`, `switch_case`, `new_case`, `save`, `upload_file` chặn traversal).
3. Chụp ảnh từng màn hình ở 1440px, 1024px, 390px vào `docs/screenshots/` để so sánh trước/sau.
4. Chạy `scripts/scan_secrets.py`; không commit `.env`, thêm `.gitignore` (`.env`, `uploads/`, `output/`).
5. Cập nhật `README.md` đúng thực tế (số test, cách chạy, yêu cầu Python).
6. Báo cáo cuối: bảng "Chức năng → trước/sau → trạng thái", liệt kê rõ việc nào chưa làm được và lý do.

Làm tuần tự: Phần 1 → khung layout (sidebar, topbar, design tokens, component) → từng màn hình → test. Sau mỗi bước chạy app và kiểm tra bằng trình duyệt thật, không chỉ đọc code.
