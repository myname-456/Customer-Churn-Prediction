# MOCK PROJECT – BANK CUSTOMER CHURN PREDICTION

## 1. Giới thiệu dự án

### 1.1. Bối cảnh
Trong lĩnh vực ngân hàng bán lẻ, việc giữ chân khách hàng hiện hữu có ý nghĩa quan trọng không kém so với việc thu hút khách hàng mới.  
Khách hàng rời bỏ dịch vụ (customer churn) không chỉ làm giảm doanh thu mà còn phản ánh các vấn đề về trải nghiệm, chi phí, hoặc mức độ gắn kết với ngân hàng.

Dự án này mô phỏng một bài toán **Customer Churn Prediction** trong bối cảnh ngân hàng, sử dụng dữ liệu lịch sử giao dịch và trạng thái tài khoản để:
- Phân tích hành vi khách hàng
- Xây dựng nhãn churn
- Chuẩn bị dữ liệu cho mô hình Machine Learning

---

### 1.2. Mục tiêu
Mục tiêu của dự án bao gồm:
- Hiểu rõ dữ liệu vận hành của khách hàng ngân hàng
- Xây dựng nhãn churn **không có sẵn trong dữ liệu gốc**
- Chuẩn bị pipeline dữ liệu đúng chuẩn cho bài toán Machine Learning
- Đảm bảo tránh data leakage và giữ tính giải thích của mô hình

---

## 2. Tổng quan dữ liệu

### 2.1. Các bảng dữ liệu sử dụng

Dự án sử dụng bốn bảng dữ liệu chính:

- `customers.csv`  
  Thông tin định danh và nhân khẩu học cơ bản của khách hàng.

- `accounts.csv`  
  Thông tin tài khoản ngân hàng của khách hàng (1 khách hàng có thể có nhiều tài khoản).

- `transactions.csv`  
  Lịch sử giao dịch phát sinh trên các tài khoản.

- `balance_snapshots.csv`  
  Snapshot số dư tài khoản tại các thời điểm khác nhau, dùng để theo dõi trạng thái và xu hướng số dư.

---

### 2.2. Đặc điểm quan trọng của dữ liệu
- Dữ liệu **không chứa sẵn nhãn churn**
- Churn cần được **suy ra từ hành vi và trạng thái tài chính**
- Dữ liệu có yếu tố thời gian → cần xử lý cẩn thận để tránh sử dụng thông tin tương lai

---

## 3. Vị trí của bước gán nhãn churn trong pipeline ML

Pipeline tổng thể của dự án được thiết kế như sau:

Raw Data
↓
Signal Extraction
↓
Churn Labeling
↓
Feature Engineering
↓
Model Training & Evaluation

---

Notebook xây dựng churn tập trung vào **bước gán nhãn (labeling)**, đóng vai trò tạo biến mục tiêu (`target variable`) cho các bước Machine Learning phía sau.

---

## 4. Định nghĩa churn trong dự án

Trong phạm vi dự án này, **churn** được hiểu là:

> Khả năng khách hàng sẽ ngừng hoặc giảm đáng kể việc sử dụng dịch vụ ngân hàng trong tương lai gần.

Lưu ý:
- Churn **không phải là dữ liệu gốc**
- Churn là **khái niệm phân tích**, được xây dựng dựa trên các giả định nghiệp vụ
- Nhãn churn chỉ phản ánh **nguy cơ rời bỏ**, không phải hành vi đã xảy ra chắc chắn

---

## 5. Phương pháp xây dựng nhãn churn

### 5.1. Cách tiếp cận
Dự án áp dụng phương pháp **rule-based churn scoring**, thay vì sử dụng một điều kiện cứng duy nhất.

Quy trình gồm các bước:
1. Trích xuất các tín hiệu hành vi và tài chính của khách hàng
2. Gán trọng số cho từng tín hiệu dựa trên giả định nghiệp vụ
3. Tính điểm churn (`churn_score`)
4. Chuyển đổi sang xác suất churn
5. Phân loại churn / non-churn theo ngưỡng

---

### 5.2. Các tín hiệu dùng để xác định churn

Các tín hiệu chính bao gồm:

- **Hoạt động giao dịch**
  - Thời gian kể từ giao dịch gần nhất (`days_since_last_txn`)

- **Trạng thái và xu hướng số dư**
  - Số dư tại thời điểm chốt (`balance_as_of`)
  - Tỷ lệ giảm số dư trong 3 tháng (`balance_drop_3m`)

- **Mức độ gắn kết sản phẩm**
  - Số lượng sản phẩm/tài khoản đang sử dụng (`num_products`)

- **Mức độ gắn bó theo thời gian**
  - Thời gian sử dụng dịch vụ (`tenure_months`)

---

## 6. Tham số và giả định nghiệp vụ

Các tham số trong bước churn labeling được khai báo dưới dạng cấu hình, bao gồm:
- Trọng số cho từng tín hiệu
- Các ngưỡng và giới hạn để kiểm soát ảnh hưởng cực đoan
---
1) Trọng số (CHURN_WEIGHTS)

- days_since_last_txn = 0.006
  Ý nghĩa : Số ngày kể từ giao dịch gần nhất
  Tác động: Lâu không giao dịch → nguy cơ churn tăng (tích lũy theo thời gian)
  Ghi chú : Hệ số nhỏ vì biến có giá trị lớn (hàng trăm ngày)

- balance_drop_3m = 1.20
  Ý nghĩa : Tỷ lệ giảm số dư trong 3 tháng gần nhất
  Tác động: Giảm số dư → churn tăng mạnh
  Ghi chú : Tín hiệu hành vi chủ động (rút tiền)

- low_balance = 0.90
  Ý nghĩa : Cờ số dư thấp (balance < LOW_BALANCE_THRESHOLD)
  Tác động: Số dư thấp → churn tăng
  Ghi chú : Phản ánh mức độ ràng buộc tài chính thấp

- multi_product_bonus = -0.55
  Ý nghĩa : Khách hàng có từ 2 tài khoản trở lên
  Tác động: Nhiều sản phẩm → churn giảm
  Ghi chú : Yếu tố bảo vệ (protective factor)

- tenure_months = -0.003
  Ý nghĩa : Thời gian gắn bó (tháng)
  Tác động: Gắn bó lâu → churn giảm nhẹ
  Ghi chú : Ảnh hưởng dài hạn, không triệt tiêu churn


2) Ngưỡng & giới hạn (THRESHOLDS / CAPS)

- LOW_BALANCE_THRESHOLD = 2,000,000
  Ý nghĩa : Ngưỡng xác định số dư thấp
  Ghi chú : Điều chỉnh theo phân phối balance khi EDA

- TENURE_CAP = 240
  Ý nghĩa : Giới hạn tenure tối đa (tháng)
  Ghi chú : Tránh khách rất lâu năm chi phối churn_score

- DAYS_CAP = 999
  Ý nghĩa : Giới hạn days_since_last_txn
  Ghi chú : Tránh giá trị cực đoan làm lệch score

- CHURN_THRESHOLD = 0.60
  Ý nghĩa : Ngưỡng xác suất phân loại churn / no churn
  Ghi chú : Điều chỉnh để đạt churn rate hợp lý
---
Các tham số này:
- Được thiết kế để **dễ điều chỉnh trong quá trình EDA**
- Không phải kết quả huấn luyện của mô hình Machine Learning
- Chỉ đóng vai trò tạo nhãn ban đầu

Việc điều chỉnh nhãn churn được thực hiện thông qua **thay đổi tham số**, không phải bằng cách can thiệp vào mô hình ML.

---

## 7. Phòng tránh data leakage

Do dữ liệu có yếu tố thời gian, dự án áp dụng các biện pháp sau để tránh data leakage:

- Xác định rõ mốc quan sát (`AS_OF_DATE`)
- Lọc toàn bộ giao dịch và snapshot số dư **không vượt quá mốc này**
- Tái tính toán các chỉ số hành vi sau khi lọc dữ liệu
- Chỉ kiểm tra và sử dụng các biến được tính từ dữ liệu quá khứ

Việc này đảm bảo rằng nhãn churn phản ánh đúng thông tin có thể quan sát tại thời điểm thực tế.

---

## 8. Kiểm tra tính hợp lý của nhãn churn (Sanity Check)

### 8.1. Mục đích
Bước sanity check nhằm:
- Đánh giá chất lượng nhãn churn
- Xác nhận các giả định nghiệp vụ đã được phản ánh đúng trong dữ liệu
- Đảm bảo nhãn có thể sử dụng an toàn cho Machine Learning

---

### 8.2. Nguyên tắc kiểm tra
Một nhãn churn hợp lý khi:
- Nhóm churn thể hiện hành vi “xấu hơn” nhóm non-churn
- Không tồn tại các giá trị phi logic (ví dụ: thời gian âm)

---

### 8.3. Các kỳ vọng kiểm tra chính

1. days_since_last_txn – Thời gian kể từ giao dịch gần nhất

Ý nghĩa nghiệp vụ
Biến này đo lường mức độ hoạt động gần đây của khách hàng.
Càng lâu không phát sinh giao dịch, khả năng khách hàng đã ngừng sử dụng dịch vụ càng cao.

Kỳ vọng khi kiểm tra

Nhóm churn = 1 có giá trị cao hơn rõ rệt

Nhóm churn = 0 có giá trị thấp hơn

Diễn giải hợp lý

Khách hàng đã lâu không giao dịch có khả năng cao đã chuyển sang ngân hàng khác hoặc không còn sử dụng tài khoản.

Dấu hiệu bất thường cần cảnh báo

Giá trị âm → data leakage (dùng giao dịch sau mốc quan sát)

churn = 1 nhưng days_since_last_txn rất thấp → logic churn sai

2. balance_as_of – Số dư tài khoản tại thời điểm chốt

Ý nghĩa nghiệp vụ
Phản ánh mức độ ràng buộc tài chính hiện tại giữa khách hàng và ngân hàng.
Số dư càng cao, chi phí rời bỏ càng lớn.

Kỳ vọng khi kiểm tra

Nhóm churn = 1 có số dư thấp hơn

Nhóm churn = 0 có số dư cao hơn

Diễn giải hợp lý

Khách hàng có ít tiền trong tài khoản thường ít ràng buộc và dễ đóng tài khoản hơn.

Dấu hiệu bất thường cần cảnh báo

Nhóm churn có số dư cao hơn non-churn → cần xem lại ngưỡng LOW_BALANCE_THRESHOLD hoặc trọng số

3. balance_drop_3m – Tỷ lệ giảm số dư trong 3 tháng gần nhất

Ý nghĩa nghiệp vụ
Đo lường xu hướng rút tiền gần đây, là tín hiệu hành vi chủ động.
Khác với số dư thấp (trạng thái), đây là biến động theo thời gian.

Kỳ vọng khi kiểm tra

Nhóm churn = 1 có giá trị dương và lớn hơn

Nhóm churn = 0 gần 0 hoặc âm (ổn định/tăng)

Diễn giải hợp lý

Việc rút tiền trong thời gian gần thường là dấu hiệu chuẩn bị rời bỏ ngân hàng.

Dấu hiệu bất thường cần cảnh báo

Hai nhóm có giá trị gần giống nhau → biến chưa có tác dụng

Non-churn giảm mạnh hơn churn → logic churn yếu

4. num_products – Số lượng sản phẩm đang sử dụng (proxy)

Ý nghĩa nghiệp vụ
Đại diện cho mức độ gắn kết (stickiness) của khách hàng với ngân hàng.
Trong dữ liệu hiện tại, số sản phẩm được xấp xỉ bằng số tài khoản.

Kỳ vọng khi kiểm tra

Nhóm churn = 1 có ít sản phẩm hơn

Nhóm churn = 0 có nhiều sản phẩm hơn

Diễn giải hợp lý

Khách hàng sử dụng nhiều sản phẩm sẽ có chi phí chuyển đổi cao hơn và ít rời bỏ hơn.

Dấu hiệu bất thường cần cảnh báo

Hai nhóm gần như không khác nhau → biến này ít đóng góp cho nhãn

5. tenure_months – Thời gian gắn bó với ngân hàng

Ý nghĩa nghiệp vụ
Phản ánh mức độ trung thành tích lũy theo thời gian.
Đây là yếu tố bảo vệ dài hạn, không phải yếu tố quyết định tức thời.

Kỳ vọng khi kiểm tra

Nhóm churn = 1 có tenure ngắn hơn

Nhóm churn = 0 có tenure dài hơn

Diễn giải hợp lý

Khách hàng gắn bó lâu năm thường quen với hệ thống và ít rời bỏ hơn, trừ khi có sự kiện đặc biệt.

Dấu hiệu bất thường cần cảnh báo

Tenure không khác biệt nhiều → không phải lỗi nghiêm trọng

Vì đây là yếu tố bảo vệ yếu
6. Tổng kết: 
- `days_since_last_txn`: churn > non-churn  
- `balance_as_of`: churn < non-churn  
- `balance_drop_3m`: churn > non-churn  
- `num_products`: churn < non-churn  
- `tenure_months`: churn < non-churn  

Nếu nhiều chỉ số đi ngược kỳ vọng, cần xem xét điều chỉnh lại tham số churn scoring.

---

## 9. Kết luận giai đoạn xây dựng churn

Sau khi xây dựng và kiểm tra:
- Nhãn churn được xác định từ dữ liệu thô, không có sẵn
- Các giả định nghiệp vụ được phản ánh hợp lý
- Không phát hiện data leakage sau khi xử lý
- Dữ liệu sẵn sàng cho giai đoạn feature engineering và modeling

Nhãn churn này sẽ được sử dụng làm biến mục tiêu cho các mô hình Machine Learning trong các bước tiếp theo của dự án.

---
Dataset cho dự án: https://drive.google.com/drive/folders/1mZICbVgmzU1PE-f7x0pxuUYNP6-SIvVI?usp=drive_link
