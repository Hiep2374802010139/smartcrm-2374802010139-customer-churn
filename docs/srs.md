# SRS — Luồng L10: Tự động phân loại yêu cầu bảo hành (Track AI)

**Học phần:** Chuyên đề tốt nghiệp 1 — HK1, NH 2026–2027
**Sinh viên:** Lê Hoàng Hiệp — MSSV 2374802010139
**Case study:** Smart CRM — Mekong Mobile

---

## 1.1. Giới thiệu, phạm vi và bảng thuật ngữ

### Bối cảnh
Mekong Mobile hiện chưa có hệ thống CRM. Yêu cầu bảo hành được ghi trên phiếu giấy; mô tả lỗi do khách kể được nhân viên tiếp nhận ghi tự do bằng chữ (V8 trong Bảng 2.2 của case study). Vì không phân nhóm được, công ty không thống kê được nguyên nhân bảo hành phổ biến để đàm phán với nhà cung cấp. Luồng L10 giải quyết vấn đề này.

### Phạm vi (một câu)
Xây dựng mô-đun tự động phân loại yêu cầu bảo hành: từ mô tả lỗi tự do của khách, hệ thống đề xuất nhóm sự cố và mức ưu tiên mặc định, cho phép nhân viên tiếp nhận chấp nhận hoặc điều chỉnh, ghi log dự đoán để phục vụ audit và đánh giá chất lượng mô hình.

### Mức WON'T (chủ ý KHÔNG làm trong học phần)
- Không huấn luyện lại mô hình trực tuyến (online learning).
- Không xử lý ảnh chụp thiết bị, chỉ xử lý văn bản mô tả lỗi.
- Không đa ngôn ngữ — chỉ tiếng Việt.
- Không tự động gán mức ưu tiên thay nhân viên; chỉ đề xuất.

### Bảng thuật ngữ

| Thuật ngữ | Định nghĩa | Tên kỹ thuật |
|---|---|---|
| Khách hàng | Cá nhân đã mua sản phẩm hoặc dùng dịch vụ của Mekong Mobile. | customer |
| Thiết bị | Máy cụ thể mà khách sở hữu, xác định bằng serial/IMEI. | device |
| Phiếu bảo hành | Yêu cầu bảo hành/sửa chữa, có mã duy nhất và vòng đời trạng thái. | ticket |
| Mô tả lỗi | Văn bản tự do do khách kể, nhân viên ghi lại trên phiếu. | issue_desc |
| Nhóm sự cố | Phân loại nguyên nhân: MÀN HÌNH / PIN / SẠC / PHẦN MỀM / NƯỚC VÀO / KHÁC. | issue_category |
| Mức ưu tiên | CAO / TRUNG_BINH / THAP — quyết định hạn cam kết. | priority |
| Hạn cam kết (SLA) | Thời điểm chậm nhất phải hoàn tất phiếu, sinh theo QT-04. | due_date |
| Dự đoán | Nhãn nhóm sự cố và độ tin cậy do mô hình trả về cho một mô tả lỗi. | prediction |
| Log dự đoán | Bản ghi lưu mô tả, nhãn dự đoán, độ tin cậy, timestamp để audit. | prediction_log |

---

## 1.2. Các bên liên quan và vai trò

| Vai trò | Được làm gì | Không được làm gì |
|---|---|---|
| Nhân viên tiếp nhận | Tạo phiếu, xem đề xuất nhóm sự cố, chấp nhận hoặc điều chỉnh, lưu phiếu. | Không được sửa log dự đoán; không xem dữ liệu trung tâm khác (QT-14). |
| Quản lý trung tâm | Xem dashboard mô hình, lịch sử dự đoán của trung tâm mình, quyết định huấn luyện lại. | Không được xóa vật lý log (QT-13); không xem toàn công ty. |
| Điều phối viên (gán nhãn) | Gán nhãn đúng cho phiếu sau khi kiểm tra thiết bị thực tế; nhãn này là ground truth cho huấn luyện. | Không sửa nhãn sau khi phiếu đã đóng. |
| Ban giám đốc | Xem báo cáo tổng hợp chất lượng mô hình, phân bố nhóm sự cố toàn công ty. | Không thao tác nghiệp vụ. |
| Dịch vụ mô hình (Model Service) | Nhận mô tả lỗi, trả về nhãn dự đoán và độ tin cậy. | Không truy cập trực tiếp CSDL nghiệp vụ, chỉ qua interface của lớp nghiệp vụ. |

---

## 1.3. Yêu cầu chức năng + User Story + MoSCoW

### Yêu cầu chức năng

| Mã | Yêu cầu chức năng |
|---|---|
| FR1 | Hệ thống phải tra cứu khách hàng theo số điện thoại (đã chuẩn hóa theo QT-02) và thiết bị theo serial/IMEI; nếu chưa có thì cho phép tạo mới. |
| FR2 | Hệ thống phải tạo phiếu bảo hành mới với đầy đủ trường bắt buộc (SĐT, serial, mô tả lỗi ≥ 10 ký tự) và sinh mã phiếu duy nhất dạng BH-000123/2026. |
| FR3 | Khi nhân viên nhập mô tả lỗi, hệ thống phải gọi mô hình phân loại và hiển thị nhóm sự cố đề xuất kèm độ tin cậy trong ≤ 2 giây (P95). |
| FR4 | Hệ thống phải cho phép nhân viên Chấp nhận hoặc Điều chỉnh nhóm sự cố đề xuất trước khi lưu phiếu; nhóm sự cố cuối cùng được lưu vào `ticket.category_id`. |
| FR5 | Mọi lần dự đoán phải được ghi vào bảng `prediction_log` (mô tả, nhãn dự đoán, độ tin cậy, timestamp, người gọi). |
| FR6 | Hệ thống phải cung cấp màn hình lịch sử dự đoán có bộ lọc theo nhóm sự cố và ngưỡng độ tin cậy. |
| FR7 | Hệ thống phải cung cấp dashboard đánh giá chất lượng mô hình: Macro F1, Confusion Matrix, F1 từng lớp, phân bố nhãn tập test. |

### User Story

- **US1 — MUST** — Là nhân viên tiếp nhận, tôi muốn tra cứu khách hàng theo SĐT và thiết bị theo serial để không phải hỏi lại khách những thông tin đã có.
  → Nối V1: hồ sơ phân tán, khách phải kể lại nhiều lần.
- **US2 — MUST** — Là nhân viên tiếp nhận, tôi muốn hệ thống tự đề xuất nhóm sự cố từ mô tả lỗi khách kể, để giảm thời gian phân loại và đảm bảo nhất quán.
  → Nối V8: mô tả lỗi ghi tự do, không phân nhóm được.
- **US3 — MUST** — Là nhân viên tiếp nhận, tôi muốn chấp nhận hoặc điều chỉnh đề xuất trước khi lưu phiếu, để không bị mô hình "áp đặt" khi nhãn sai.
  → Nối V8 + tránh sai sót khi nhãn nhiễu (~85% đồng thuận).
- **US4 — MUST** — Là quản lý trung tâm, tôi muốn mọi dự đoán được ghi log kèm mô tả và độ tin cậy, để có bằng chứng khi cần audit và cải tiến mô hình.
  → Nối NFR3, NFR5.
- **US5 — SHOULD** — Là quản lý trung tâm, tôi muốn xem lịch sử dự đoán lọc theo nhóm và độ tin cậy thấp, để phát hiện các trường hợp mô hình yếu và bổ sung dữ liệu huấn luyện.
  → Nối NFR2.
- **US6 — SHOULD** — Là quản lý trung tâm, tôi muốn xem dashboard Macro F1 và Confusion Matrix, để biết khi nào cần huấn luyện lại mô hình.
  → Nối NFR2 và NFR5.

---

## 1.4. Yêu cầu phi chức năng

| Mã | NFR | Ngưỡng đo được |
|---|---|---|
| NFR1 | Hiệu năng dự đoán | Thời gian phản hồi của endpoint `/predict` cho mô tả lỗi ≤ 500 ký tự: P50 ≤ 500 ms, P95 ≤ 2 giây. |
| NFR2 | Chất lượng mô hình | Macro F1 ≥ 0,70 trên tập test; F1 lớp KHÁC ≥ 0,55 (lớp thiểu số, 5,5%). |
| NFR3 | Bảo mật dữ liệu | Không ghi `issue_desc` ra file log ngoài; 100% log dự đoán ghi vào bảng `prediction_log` có kiểm soát quyền. |
| NFR4 | Khả năng bảo trì | Thêm 1 nhóm sự cố mới chỉ cập nhật dữ liệu + bảng `issue_category`, không sửa mã nguồn API (0 dòng code API thay đổi). |
| NFR5 | Tái lập | Cùng `random_seed=42`, `test_size=0.15`, `max_features=20000`: kết quả Macro F1 sai khác ≤ 0,001. |
| NFR6 | Khả dụng | Trong giờ hành chính (8h–18h), mô-đun phân loại đáp ứng ≥ 99% request; downtime dự kiến ≤ 4 giờ/tháng. |

---

## 1.5. Ràng buộc và quy tắc nghiệp vụ

| Mã | Quy tắc | Nguồn |
|---|---|---|
| QT-02 | SĐT được chuẩn hóa về dạng `0xxxxxxxxx` trước khi lưu. | Bảng 9.1 case study |
| QT-04 | Hạn cam kết sinh tự động từ thời điểm tiếp nhận: CAO = 24h, TRUNG_BINH = 72h, THAP = 120h, chỉ tính ngày làm việc T2–T7. | Bảng 9.1 |
| QT-06 | Phiếu chỉ chuyển trạng thái theo vòng đời Hình 6.2; mọi chuyển trạng thái ghi vào `ticket_status_log`. | Bảng 9.1 |
| QT-11 | Chống rò rỉ dữ liệu: nhãn `category_id` được gán SAU khi phiếu xử lý xong → KHÔNG được dùng làm feature. Mọi đặc trưng phải tính từ dữ liệu có trước thời điểm tạo phiếu. | Bảng 9.1 |
| QT-13 | Không xóa vật lý phiếu/log; chỉ soft delete và giữ lịch sử. | Bảng 9.1 |
| QT-14 | Nhân viên chỉ xem dữ liệu của trung tâm mình; quản lý xem trong đơn vị mình phụ trách. | Bảng 9.1 |

---

## 1.6. Bảng truy vết yêu cầu

| FR | User Story | Use Case | MoSCoW | Bảng dữ liệu | Màn hình |
|---|---|---|---|---|---|
| FR1 | US1 | UC01 Tra cứu khách hàng & thiết bị | MUST | customer, device | M1 |
| FR2 | US1, US2 | UC02 Tạo phiếu bảo hành | MUST | ticket | M1 |
| FR3 | US2 | UC03 Dự đoán nhóm sự cố | MUST | prediction_log, issue_category | M1 |
| FR4 | US3 | UC03 Dự đoán nhóm sự cố | MUST | ticket.category_id | M1 |
| FR5 | US4 | UC04 Ghi log dự đoán | MUST | prediction_log | M1 |
| FR6 | US5 | UC05 Tra cứu lịch sử dự đoán | SHOULD | prediction_log | M2 |
| FR7 | US6 | UC06 Đánh giá mô hình | SHOULD | model_metric_log | M3 |