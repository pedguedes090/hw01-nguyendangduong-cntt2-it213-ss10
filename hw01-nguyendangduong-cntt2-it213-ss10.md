# HW01 — Phân Tích & Lựa Chọn Giải Pháp Triển Khai Langfuse

**Học viên:** Nguyễn Đăng Dương — **Lớp:** CNTT2 — **Bài:** SS10 — **HW01**

**Link GitHub:** https://github.com/pedguedes090/hw01-nguyendangduong-cntt2-it213-ss10.git

---

## 1. Bảng so sánh chi tiết 3 phương án triển khai

| Tiêu chí | **Phương án A** — Self-Host tối giản (chỉ PostgreSQL) | **Phương án B** — Self-Host đầy đủ (PostgreSQL + ClickHouse) | **Phương án C** — Self-Host + PostgreSQL External (AWS RDS / Postgres vật lý) |
|---|---|---|---|
| **Kiến trúc tổng quan** | 1 container Langfuse + 1 container PostgreSQL trên cùng Docker Compose | Docker Compose gồm: Langfuse (web + worker) + PostgreSQL (metadata) + ClickHouse (analytics) + Redis (queue) | Docker Compose chỉ chứa Langfuse; PostgreSQL nằm ngoài Docker (RDS / máy vật lý), kết nối qua mạng riêng/VPC |
| **Bảo mật dữ liệu (Data Privacy)** | Thấp: dữ liệu nằm trong container local, ít kiểm soát encryption-at-rest, không có cơ chế bảo vệ chuyên nghiệp (KMS, IAM, audit) | Trung bình: vẫn chạy DB trong container local; rò rỉ dữ liệu ra volume Docker; chưa kiểm soát truy cập mức doanh nghiệp | **Cao nhất:** kế thừa toàn bộ bảo mật của dịch vụ quản lý (encryption-at-rest, TLS, IAM/security group, audit log, VPC private subnet, hỗ trợ KMS + CMK do doanh nghiệp tự quản lý) |
| **Chi phí & tài nguyên (CPU/RAM/Storage)** | **Thấp nhất:** chỉ cần ~1-2 GB RAM, 1 vCPU; không cần ClickHouse (ClickHouse ngốn 4-8 GB RAM) | Cao: ClickHouse + Redis thêm ~4-6 GB RAM; phải trả thêm chi phí vận hành 2 DB engine | Chi phí server nhỏ nhưng phát sinh phí thuê RDS (giá theo instance) hoặc chi phí hạ tầng Postgres vật lý; bù lại giảm chi phí vận hành/backup so với tự làm |
| **Độ phức tạp triển khai** | **Đơn giản nhất:** 1 file compose 2 services; phù hợp dev local | Trung bình–cao: phải cấu hình song song Postgres + ClickHouse + Redis, đồng bộ schema, migration, tuning ClickHouse | Trung bình: compose chỉ 1 service Langfuse nhưng phải xử lý network/VPC, security group, kết nối an toàn, IAM role, SSL |
| **Khả năng sao lưu/phục hồi (Backup & Recovery)** | Thấp: phải tự viết script backup; dễ mất dữ liệu nếu volume hỏng; RPO/RTO tệ | Trung bình: phải backup riêng 2 DB + đảm bảo nhất quán giữa Postgres (metadata) và ClickHouse (analytics) — phức tạp khi khôi phục điểm thời điểm | **Cao nhất:** RDS tự động snapshot (PITR), multi-AZ, automatic failover; doanh nghiệp có thể áp dụng chiến lược backup chuẩn (RPO phút, RTO giờ) |
| **Phù hợp vòng đời** | Phát triển local / demo / POC | Môi trường staging, scale vừa, cần analytics mạnh nhưng chấp nhận tự vận hành | Production / quy mô doanh nghiệp, yêu cầu tuân thủ, dữ liệu phải nằm trong hạ tầng quản lý của công ty |
| **Downsides lớn nhất** | Không mở rộng được khi volume trace tăng; mất dữ liệu khi container bị xóa | ClickHouse là single point; chi phí vận hành cao; backup 2 DB phức tạp | Chi phí thuê RDS; phụ thuộc kết nối mạng; triển khai ban đầu chậm hơn do thiết lập hạ tầng |

---

## 2. Lựa chọn tối ưu cho RikkeiPay: **Phương án C — Langfuse Self-Host + PostgreSQL External (AWS RDS)**

### 2.1. Lý do lựa chọn

1. **An toàn thông tin khách hàng là yêu cầu bất biến.** RikkeiPay xử lý giao dịch tài chính thực — trace Langfuse chứa PII, số tài khoản, số tiền giao dịch. Khi DB nằm trong container Docker local (phương án A/B), dữ liệu chỉ được bảo vệ bằng volume thường không có mã hóa ở mức disk, không có audit trail, không có kiểm soát truy cập cấp doanh nghiệp. Với AWS RDS, dữ liệu được mã hóa at-rest (KMS), mã hóa in-transit (TLS), cô lập bằng security group/VPC private subnet — đáp ứng chuẩn bảo mật ngành tài chính (VD: yêu cầu của NHNN về bảo vệ dữ liệu khách hàng).
2. **Sao lưu & phục hồi chuyên nghiệp mà không tốn công vận hành.** RDS tự động snapshot hàng ngày + PITR (khôi phục đến từng phút), hỗ trợ multi-AZ failover → downtime tối thiểu, RPO/RTO đạt chuẩn doanh nghiệp. Phương án A/B muốn đạt mức này phải tự xây dựng và duy trì hệ thống backup — vừa tốn công vừa dễ sai sót, nguy hiểm với dữ liệu tài chính.
3. **Giảm thiểu downtime (yêu cầu đề bài).** Dữ liệu tài chính không thể chấp nhận mất mát khi một container bị xóa nhầm (`docker compose down -v` xóa luôn volume). RDS với automatic failover + snapshot giúp khôi phục nhanh; bản thân container Langfuse stateless nên có thể scale/restart thoải mái.
4. **Tách biệt trách nhiệm (separation of concerns).** Langfuse (tầng ứng dụng, stateless) và PostgreSQL (tầng dữ liệu, stateful) tách rời — cập nhật Langfuse không ảnh hưởng dữ liệu, backup chỉ tập trung vào một nơi duy nhất.
5. **Chi phí hợp lý cho quy mô production.** RDS db.t3.small/medium phù hợp hàng ngàn trace/ngày; chi phí thuê thấp hơn nhiều so với chi phí nhân công vận hành + rủi ro mất dữ liệu của phương án tự host DB.

### 2.2. Khi nào nên cân nhắc phương án khác

- Nếu chỉ là **môi trường dev/POC** → phương án A là đủ, tiết kiệm tài nguyên.
- Nếu cần **analytics/aggregation tốc độ cao trên hàng triệu trace** → kết hợp thêm ClickHouse (nâng cấp từ C thành C + ClickHouse managed).

---

## 3. Nhược điểm / rủi ro lớn nhất của các phương án bị loại bỏ

### Phương án A (bị loại)

- **Không mở rộng được:** PostgreSQL đơn lẻ chết nghẽn khi trace tăng cao; không có ClickHouse nên các truy vấn analytics (cost/latency tổng hợp) chậm và tốn tài nguyên.
- **Rủi ro mất dữ liệu nghiêm trọng:** toàn bộ dữ liệu nằm trong volume container; xóa container/volume, hỏng ổ đĩa local → mất sạch trace tài chính mà không có cách khôi phục. Đây là rủi ro **lớn nhất** — không chấp nhận được với hệ thống ngân hàng.
- **Bảo mật kém:** không mã hóa dữ liệu, không kiểm soát truy cập, dễ bị lộ PII.

### Phương án B (bị loại)

- **Chi phí vận hành cao:** phải vận hành song song 3 hệ thống (Postgres + ClickHouse + Redis) — mỗi hệ là một nguồn sự cố tiềm ẩn; ClickHouse yêu cầu RAM lớn (4-8 GB), không phù hợp nếu hạ tầng hiện tại của RikkeiPay còn hạn chế.
- **Backup 2 DB không nhất quán:** metadata nằm ở Postgres, analytics nằm ở ClickHouse; backup riêng lẻ dễ dẫn đến trạng thái lệch thời điểm khi khôi phục → dữ liệu không đồng bộ giữa trace và metrics.
- **Single point of failure:** nếu container ClickHouse crash, toàn bộ phần analytics chết, kéo theo hạ tầng giám sát sụp đổ ngay khi cần nhất.

---

## 4. Kết luận

- **Chọn Phương án C** cho môi trường production của RikkeiPay: an toàn dữ liệu, backup/RPO-RTO chuẩn doanh nghiệp, downtime tối thiểu, chi phí vận hành hợp lý.
- Phương án A chỉ dùng cho dev local; phương án B chỉ hợp lý khi cần scale analytics mạnh và chấp nhận chi phí vận hành — đều không đáp ứng được yêu cầu bảo mật và độ tin cậy của hệ thống giao dịch tài chính.
