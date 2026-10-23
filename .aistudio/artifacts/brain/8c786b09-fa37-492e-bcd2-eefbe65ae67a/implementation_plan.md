# Kế Hoạch Ổn Định Dữ Liệu & Khắc Phục Lỗi Trắc Nghiệm Theo Unit

Ứng dụng **Dental English Pro** cần một giải pháp toàn diện để đảm bảo khi chọn Unit trong phần trắc nghiệm, câu hỏi chỉ xuất hiện trong Unit đã chọn; đồng thời bảo tồn 100% tiến trình học tập của người dùng khi bấm F5 hoặc khi đẩy mã nguồn mới lên GitHub.

---

### User Review & Critical Decisions

> [!IMPORTANT]
> Dưới đây là các quyết định kỹ thuật đã được thống nhất:
> - **Lựa chọn đáp án nhiễu (Distractors):** Cho phép lấy từ vựng trong cùng Unit hoặc khác Unit, nhưng bắt buộc cùng từ loại (danh từ/động từ/tính từ) và có độ dài/ngữ cảnh tương đương để tối đa hóa khả năng gây nhiễu học thuật.
> - **Khôi phục tiến trình khi F5:** Lưu trữ chính xác vị trí câu hỏi (`quizIndex`), điểm số hiện tại (`quizScore`), và danh sách câu hỏi đã xáo trộn của Unit đang chọn để khi F5 người dùng tiếp tục ngay tại câu đang làm.
> - **Chiến lược khóa lưu trữ (Storage Key Strategy):** Bỏ các hậu tố phiên bản cứng trong tên khóa (chuyển sang các khóa vĩnh viễn như `dental_pro_specialties`, `dental_pro_progress`), đồng thời tích hợp cơ chế **Tự động chuyển đổi dữ liệu (Auto-Migration)** từ phiên bản `_v7_2` sang khóa mới mà không làm mất bất kỳ từ vựng hay điểm SRS nào.
> - **Cơ chế cập nhật phiên bản mới (What's New):** Khóa `dental_pro_last_seen_version` sẽ so sánh với `APP_BUILD_VERSION`. Khi có phiên bản mới trên GitHub, modal giới thiệu tính năng mới sẽ bật đúng 1 lần và tự tắt sau khi người dùng xác nhận.

---

### 1. Overview & Core Concept

- **Mục tiêu:**
  1. Khi người dùng tick chọn 1 hoặc nhiều Unit, mảng câu hỏi trắc nghiệm lập tức được tái cấu trúc chỉ chứa các thuật ngữ thuộc các Unit đó.
  2. Bấm F5 hoặc tắt trình duyệt bất ngờ không làm mất dữ liệu học tập, câu hỏi đang làm dở hay chuỗi ngày học (Streak).
  3. Khi tác giả nâng cấp tính năng trên GitHub (ví dụ từ v7.2 lên v7.3), toàn bộ người dùng cũ giữ nguyên lịch sử SRS và nhận được bảng thông báo tính năng mới.

---

### 2. User Experience & Visual Design

- **Trải nghiệm chọn Unit ở Drawer & Dashboard:**
  - Khi tick chọn 1 Unit (ví dụ Unit 2), thanh tiêu đề trắc nghiệm lập tức cập nhật: `Câu 1 / 29 (Unit 2)`.
  - Không còn hiện tượng các câu hỏi của Unit 1, 3, 4, 5 nhảy xen kẽ.
  - Khi chuyển câu, thứ tự và tiến độ hiển thị trực quan và mượt mà.
- **Trải nghiệm khi F5:**
  - Nếu đang ở Tab Trắc nghiệm câu số 5 với 4 câu đúng, khi nhấn F5 trang sẽ tải lại đúng Tab Trắc nghiệm, đúng câu số 5 và điểm số `Đúng: 4` được bảo toàn nguyên vẹn.
- **Trải nghiệm cập nhật phiên bản:**
  - Người dùng truy cập phiên bản mới sẽ thấy hộp thoại Modal "Bản Cập Nhật Mới" xuất hiện 1 lần duy nhất để nắm bắt các cải tiến, bấm "Đã Hiểu" là hộp thoại được ghi nhận và không làm phiền lại.

---

### 3. Key Product Decisions & Trade-Offs

- **Quyết định 1: Cơ chế làm mới `quizPool` khi thay đổi bộ lọc:**
  - *Phương án:* Viết lại hàm `refreshQuizPool(keepPositionIfValid)` được gọi đồng bộ từ mọi nguồn thay đổi Unit (Drawer checkbox, nút "Chọn tất cả", nút "Chỉ chọn Unit 1", nút "Luyện Unit" ở Dashboard, checkbox "Ôn từ hay sai").
  - *Lý do:* Khắc phục triệt để lỗ hổng trước đây do `syncAllViews()` chỉ gọi `loadQuizQuestion()` khi mảng câu hỏi chưa rỗng.
- **Quyết định 2: Tự động di chuyển dữ liệu (Data Migration Engine):**
  - *Phương án:* Kiểm tra nếu khóa vĩnh viễn mới chưa có dữ liệu nhưng khóa cũ `_v7_2` đang có dữ liệu thì tự động đọc dữ liệu cũ, ghi sang khóa mới và hợp nhất thêm các từ mới trong phiên bản hiện tại.
  - *Lý do:* Tránh 100% rủi ro mất dữ liệu học viên khi cập nhật code trên GitHub.
- **Quyết định 3: Thêm sự kiện `beforeunload`:**
  - *Phương án:* Đăng ký `window.addEventListener('beforeunload', ...)` để tự động flush toàn bộ biến trạng thái xuống `localStorage` trước khi tab bị đóng hoặc F5.

---

### 4. Technical Architecture & Data Strategy

```
┌─────────────────────────────────────────────────────────────┐
│                    User Actions (UI Layer)                  │
│  - Tick chọn Unit / Bấm 'Luyện Unit'                       │
│  - Trả lời trắc nghiệm / Lật flashcard                      │
│  - Bấm F5 hoặc đóng trình duyệt                             │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                 Event Synchronization Engine                │
│  - refreshQuizPool(activeUnits, filterMistakes)             │
│  - recordStudyOutcome() & saveStudyProgress()               │
│  - window.addEventListener('beforeunload')                  │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                Persistent Storage & Migration               │
│  - Check migration from legacy keys (v7_2 -> stable)       │
│  - Stable Key: 'dental_pro_specs' (Vocab, SRS, Mistakes)    │
│  - Stable Key: 'dental_pro_settings' (Tab, Quiz state)      │
│  - Stable Key: 'dental_pro_seen_version' (What's New Modal) │
└─────────────────────────────────────────────────────────────┘
```

#### Các bước triển khai chi tiết:
1. **Chuẩn hóa danh mục Storage Keys:**
   - Thay thế các key chứa hậu tố `_v7_2` thành khóa chuẩn cố định.
   - Viết hàm `migrateLegacyData()` chạy một lần khi khởi tạo `initDatabase()`.
2. **Sửa lỗi lọc Unit trong phần Trắc Nghiệm:**
   - Tạo hàm `buildQuizPoolForUnits(unitsSet, onlyMistakes)` lọc chính xác tập từ vựng hợp lệ.
   - Cập nhật các sự kiện thay đổi Unit ở Drawer, nút tắt Dashboard, nút chuyển câu và nút đảo câu để luôn đồng bộ lại `quizPool`.
   - Nâng cấp thuật toán `getDistinctDistractors()` đảm bảo 3 đáp án sai cùng từ loại, có tính tương đồng cao để gây nhiễu tốt.
3. **Bảo toàn tiến trình khi F5:**
   - Lưu trữ và đối chiếu ID câu hỏi trong `restoreStudyProgress()`.
   - Bổ sung listener `beforeunload` để đảm bảo dữ liệu ghi xuống đĩa trước khi trình duyệt tải lại.
4. **Kiểm tra và kiểm thử:**
   - Thử nghiệm chọn riêng từng Unit (Unit 1, Unit 2, Unit 3, Unit 4, Unit 5) và xác nhận 100% câu hỏi thuộc đúng Unit đó.
   - Thử nghiệm F5 giữa chừng phiên làm bài để đảm bảo khôi phục chính xác câu và điểm.
   - Chạy `compile_applet` và `lint_applet` để xác nhận ứng dụng build hoàn hảo.
