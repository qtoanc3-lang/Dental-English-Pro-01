# Kế Hoạch Khắc Phục Lỗi QA/QC — Dental English Pro v7.2.1

## 1. Tổng Quan Mục Tiêu
Triển khai bản vá toàn diện cho **Dental English Pro** (nâng mã phiên bản lên `v7.2.1_QA_PATCH`) nhằm xử lý triệt để toàn bộ **9 lỗi trong Báo cáo Kiểm thử & Rà soát Lỗi (QA/QC Report)** theo đúng 3 nhóm mức độ nghiêm trọng, đồng thời đồng bộ danh sách model Gemini chính thức sang cả **Tab 7 (Quét PDF Tạo Đề Thi)** và tự động kích hoạt **Unit 5** cho người dùng nâng cấp từ phiên bản cũ.

---

## 2. Chi Tiết Các Hạng Mục Sửa Lỗi (9/9 Lỗi QA/QC)

### 🔴 Nhóm 1: Lỗi Nghiêm Trọng (Critical Bugs)
1. **Lỗi #1 — Đồng bộ đóng/mở khung "Chi tiết Unit" trên cả 3 vị trí (`toggleUnitDetail`)**:
   - **Vấn đề**: Hàm `toggleUnitDetail(unitName, btnElement)` hiện chỉ tìm `detailBox_${safeKey}` (trong Drawer), khiến thao tác bấm mở rộng ở **Tab 0 (Dashboard)** (`dashDetailBox_${safeKey}`) và **Tab 5 (Cây thư mục)** (`treeDetailBox_${safeKey}`) chỉ xoay icon mũi tên mà không bung khung nội dung.
   - **Giải pháp**: Cập nhật `toggleUnitDetail` để duyệt đồng thời cả 3 ID (`detailBox_${safeKey}`, `dashDetailBox_${safeKey}`, `treeDetailBox_${safeKey}`) và đồng bộ trạng thái `.expanded` + góc xoay mũi tên ở mọi nơi đang hiển thị.

2. **Lỗi #2 — Chuẩn hóa danh sách Model Gemini API tránh lỗi 404 Not Found**:
   - **Vấn đề**: Các model `gemini-3.8-flash` và `gemini-3.1-flash-lite` gây lỗi 404 trên endpoint `v1beta`, làm trễ 1–2 giây trước khi fallback.
   - **Giải pháp**:
     - Cập nhật danh sách model chuẩn `const models = ["gemini-2.5-flash", "gemini-2.0-flash", "gemini-1.5-flash-latest"];` tại tất cả các hàm gọi AI:
       1. `queryThreeTierAi` (Trợ lý AI 3 tầng)
       2. `enrichWordWithAiFallback` (Tự làm giàu từ vựng đơn & hàng loạt ở Tab 5)
       3. `btnTestAiKey` (Nút Thử kết nối API Key trong AI Hub)
       4. `btnDocGenerateExam` & `btnDocExtractVocab` tại **Tab 7 (Quét PDF/Slide Tạo Đề Thi)** — chuyển từ gọi đơn lẻ sang vòng lặp fallback qua chuỗi 3 model `["gemini-2.5-flash", "gemini-2.0-flash", "gemini-1.5-flash-latest"]`.
     - Cập nhật nhãn hiển thị trạng thái AI trong giao diện cho đồng bộ.

3. **Lỗi #3 — Khôi phục đầy đủ dữ liệu học tập từ cả bản `v7.1` và `v7.2` (`migrateLegacyStorage`)**:
   - **Vấn đề**: `migrateLegacyStorage()` chỉ quét hậu tố `_v7_2`, bỏ sót tiến trình SRS, lịch sử thi và chuỗi ngày học (Streak) của người dùng từ bản `_v7_1`.
   - **Giải pháp**: Bổ sung toàn bộ cặp khóa `_v7_1` và `_v7_2` vào danh sách ánh xạ di chuyển dữ liệu sang khóa lưu trữ chuẩn hiện hành.

---

### 🟠 Nhóm 2: Lỗi Logic & Trải Nghiệm Người Dùng (Medium Bugs)
4. **Lỗi #4 — Khớp chính xác từ vựng bằng Regex ranh giới từ (`\b`) khi phân tích Đề Cương (`analyzeSyllabus`)**:
   - **Vấn đề**: Dùng `.includes()` thuần túy gây nhận diện nhầm (False Positives) các từ ngắn như `"cut"`, `"wear"`, `"fill"`, `"root"` hoặc từ tiếng Việt ngắn như `"mô"`, `"răng"`, `"lợi"`, và bỏ sót các biến thể sau dấu `/` hoặc trong ngoặc đơn `(...)`.
   - **Giải pháp**:
     - Tách tất cả biến thể tiếng Anh qua dấu `/` và ngoặc đơn `(...)`.
     - Sử dụng biểu thức chính quy có ranh giới từ `new RegExp("\\b" + escaped + "s?\\b", "i")` cho các biến thể tiếng Anh có độ dài `>= 3`.
     - Đối với nghĩa tiếng Việt, loại bỏ phần chú thích trong ngoặc đơn và chỉ so khớp khi chuỗi nghĩa đủ dài (`>= 6` ký tự) để tránh khớp nhầm từ đơn âm tiết.

5. **Lỗi #5 — Sửa lỗi nút gạt "Chế độ riêng đề cương" (`chkStrictSyllabusMode`) ở Tab 6 (`btnGenerateCBT`)**:
   - **Vấn đề**: Khi bỏ tích `chkStrictSyllabusMode`, nhánh `else` vẫn đọc `cbtSyllabusInput` và lọc theo đề cương thay vì lấy toàn bộ từ của các Unit đang chọn.
   - **Giải pháp**: Chỉ lọc theo `parsedSyllabusWords` hoặc `cbtSyllabusInput` khi `isStrict === true`. Khi người dùng bỏ tích `chkStrictSyllabusMode`, lấy trực tiếp ngân hàng từ của các Unit đang chọn (`[...getFilteredVocab()]`). Đồng thời áp dụng hàm lọc chuẩn `analyzeSyllabus` cho văn bản dán tay.

6. **Lỗi #6 — Loại bỏ đáp án nhiễu đồng nghĩa trong Trắc nghiệm & CBT (`getDistinctDistractors`)**:
   - **Vấn đề**: Các cặp từ đồng nghĩa có chú thích trong ngoặc như `"Sâu răng"` (`u5_19`) và `"Sâu răng (thuật ngữ phổ thông)"` (`u5_20`), hoặc `"Lợi (Thuật ngữ thông thường)"` và `"Lợi (Thuật ngữ giải phẫu)"` cùng xuất hiện trong 1 câu hỏi.
   - **Giải pháp**: Chuẩn hóa cả `word` và `meaning` bằng `.toLowerCase().replace(/\([^)]*\)/g, "").split("/")[0].trim()` trước khi đưa vào tập kiểm tra trùng lặp `usedWords` và `usedMeanings`.

7. **Lỗi #7 — Tự động kích hoạt Unit 5 cho người dùng cũ khi nâng cấp phiên bản**:
   - **Vấn đề**: Người dùng cũ có `selectedUnits` lưu trong `localStorage` chỉ gồm Unit 1–4 nên không thấy Unit 5 khi vào học.
   - **Giải pháp**:
     - Nâng `APP_BUILD_VERSION` lên `"v7.2.1_QA_PATCH"`.
     - Trong `restoreStudyProgress()`, kiểm tra nếu phiên bản lưu trong `STORAGE_KEY_SEEN_VERSION` khác `APP_BUILD_VERSION` (hoặc nếu `"Unit 5"` có trong `getAllUnits()` khi nâng cấp), tự động bổ sung mọi Unit mới từ `getAllUnits()` vào `selectedUnits` và lưu lại.

---

### 🟡 Nhóm 3: Lỗi Nhỏ & Tối Ưu Hóa (Low / Polish)
8. **Lỗi #8 — Bảo vệ lỗi Offline khi tải thư viện PDF.js trên `<head>`**:
   - **Vấn đề**: Gọi trực tiếp `pdfjsLib.GlobalWorkerOptions.workerSrc` mà không kiểm tra `typeof window.pdfjsLib !== "undefined"`, gây lỗi ReferenceError nếu mở app offline khi chưa cache CDN.
   - **Giải pháp**: Bọc câu lệnh khởi tạo worker trong điều kiện `if (typeof window.pdfjsLib !== "undefined")` trên `<head>` và kiểm tra an toàn trước khi đọc file `.pdf` hoặc `.docx`.

9. **Lỗi #9 — Che khuyết chính xác theo ranh giới từ trong câu ví dụ lâm sàng (`maskWordInSentence`)**:
   - **Vấn đề**: Thay thế chuỗi con không có `\b` làm hỏng các từ dài chứa từ khóa ngắn bên trong.
   - **Giải pháp**: Tách tất cả biến thể `/` và `(...)`, sắp xếp giảm dần theo độ dài và thay thế bằng `new RegExp("\\b" + escaped + "\\w*", "gi")` thành `"[_____]"`.

---

## 3. Kiểm Thử & Xác Minh Sau Khi Triển Khai
- Chạy công cụ kiểm tra biên dịch (`compile_applet`) và kiểm tra cú pháp (`lint_applet`) đảm bảo 0 lỗi.
- Rà soát toàn bộ luồng đóng/mở chi tiết Unit ở Dashboard, Drawer và Cây thư mục, luồng lọc đề cương CBT, và chuỗi fallback Gemini API.
