Báo cáo LAB 17 — Data Pipeline Engineering

Họ tên: Nguyễn Thế Công · Lớp: AICB-P2T2 · Ngày: 17/08/2026
0 · Kết quả make verify
Plaintext

BẢNG                  ỔN ĐỊNH    SỐ HÀNG    KỲ VỌNG   GHI CHÚ
gold_training_set     ✓ ok        12,480     12,480   ✓
gold_feature_daily    ✓ ok         9,100      9,100   ✓
gold_doc_chunks       ✓ ok        31,200     31,200   ✓
quarantine_tickets    ✓ ok           312        312   ✓
Tổng kết: 4 / 4 tiêu chí đạt

1 · Kích thước bảng training tăng sau mỗi lần chạy

    Triệu chứng: Bảng bị lặp bản ghi, tăng gấp đôi số hàng sau mỗi lần chạy lại.

    Nguyên nhân: Thiếu unique_key và incremental_strategy, dbt mặc định dùng append.

    Khắc phục: Thêm unique_key = 'ticket_id' và incremental_strategy = 'merge' vào gold_training_set.sql.

    Bằng chứng: Sau sửa đạt 12,480 hàng, checksum ổn định tuyệt đối qua 3 lượt chạy.

2 · Bảng đặc trưng theo ngày thiếu hàng ở các ngày quá khứ

    Triệu chứng: gold_feature_daily thiếu 455 hàng (~5%) ở các ngày cũ.

    P99 độ trễ đo được: 2.73 ngày (5.05% bản ghi trễ > 1 ngày, max delay ~2.94 ngày).

    Lookback đã chọn: 3 ngày (vừa đủ bao phủ P99 mà không quét thừa tài nguyên).

    Nguyên nhân: Lọc theo where event_date > (select max...) bỏ sót dữ liệu đến muộn (late-arriving data).

    Khắc phục: Đổi điều kiện lọc thành where event_date >= (select max(event_date) - interval 3 day) kèm composite key ['event_date', 'customer_id'] và strategy delete+insert.

    Bằng chứng: Đạt chuẩn 9,100 / 9,100 hàng.

    P99 vs Max: Chọn P99 để tránh chi phí compute quá lớn khi xuất hiện ngoại lệ (outlier) trễ hàng tháng bắt pipeline phải quét lại toàn bộ lịch sử ở mỗi chu kỳ.

3 · Kiểu dữ liệu cột priority thay đổi giữa chu kỳ

    Triệu chứng: Tầng Silver có 6,606 hàng sai/NULL; mô hình ML giảm độ chính xác; quarantine_tickets có 0 hàng.

    Nguyên nhân: Backend đổi kiểu dữ liệu priority sang text (low, medium, high, urgent) làm gãy logic ép kiểu.

    Xử lý 3 nhóm:

        Chuỗi số ('1'..'4'): Ép kiểu trực tiếp về 1..4.

        Chuỗi text ('low'..'urgent'): Map tương ứng thành 1, 2, 3, 4.

        Dữ liệu rác (-1, 0, 5, 'unknown', NULL): Chuyển vào bảng quarantine_tickets (312 hàng).

    Khắc phục: Hoàn thiện macro normalize_priority, lọc bản ghi hỏng trước khi row_number() trong silver_tickets.sql, bật contract.enforced: true.

    Bằng chứng: quarantine_tickets = 312 hàng, silver_tickets không còn NULL, dbt test 9/9 pass.

    Thiết kế Dead-Letter Queue: Giữ raw data ở Bronze, lọc và cách ly ở Silver. Không để pipeline dừng đột ngột (Hard Fail) để dữ liệu hợp lệ vẫn lưu thông liên tục cho downstream.

5 · Tổng kết

    Nhiệm vụ 1: Luôn kiểm tra tính Idempotent và cấu hình unique_key + merge cho bảng incremental.

    Nhiệm vụ 2: Luôn đo phân bố độ trễ (P95/P99 latency) để đặt Lookback Window xử lý late-arriving data.

    Nhiệm vụ 3: Triển khai Data Contract và Dead-Letter Queue để cách ly lỗi mà không làm nghẽn pipeline.