
---

# **Hồ sơ Năng lực Kỹ thuật: Nền tảng Phân tích Dữ liệu Hiệu suất cao DA-ODA**

> **Tầm nhìn:** Trao quyền cho Nhà phân tích Dữ liệu bằng **Tốc độ**, **Tự động hóa** và **Sự Tin cậy**.

---

### **1. Bối cảnh: Thách thức của Kỷ nguyên Dữ liệu 2025**

Năm 2025, `pandas.read_csv()` trên một file 10GB không còn là giải pháp, đó là vấn đề. Các quy trình phân tích truyền thống đang trở thành rào cản, làm chậm quá trình ra quyết định và bào mòn niềm tin vào dữ liệu.

**DA-ODA** ra đời không phải để cải tiến, mà để **cách mạng hóa** quy trình làm việc của Nhà phân tích. Đây là câu trả lời cho bài toán về tốc độ và quy mô, một nền tảng được thiết kế có chủ đích để biến việc phân tích dữ liệu từ một gánh nặng thủ công thành một lợi thế chiến lược.

---

### **2. Tính năng Vượt trội - Những "Năng lực" Cốt lõi** 🎯

Thay vì liệt kê, hãy xem cách DA-ODA giải quyết trực tiếp những "nỗi đau" lớn nhất của ngành dữ liệu.

| "Nỗi đau" của Nhà phân tích | ✨ Giải pháp của DA-ODA: "Pain Point Killer" | 🚀 Bằng chứng Kỹ thuật |
| :--- | :--- | :--- |
| *"Chờ đợi `Pandas` load file CSV 10GB và xử lý trong tuyệt vọng."* | **Profiling Hiệu suất Cao:** Phân tích hàng triệu dòng trong vài giây, không phải vài phút. | **Polars Engine:** Tận dụng 100% CPU đa lõi, xử lý song song và tối ưu hóa bộ nhớ. |
| *"Lấy mẫu `LIMIT 10000` không đại diện cho dữ liệu thực tế."* | **Lấy mẫu Thông minh & Tối ưu:** Tự động chọn chiến lược lấy mẫu hiệu quả nhất cho từng loại CSDL. | **`DataSampler`:** Sử dụng `TABLESAMPLE` (PostgreSQL), lấy mẫu ngẫu nhiên theo dải PK (MySQL), và **ConnectorX** (Rust-based) để I/O tốc độ ánh sáng. |
| *"Tải cả bảng về chỉ để kiểm tra `NOT NULL`."* | **Engine DQ "Database-First":** Thực thi các quy tắc chất lượng dữ liệu ngay trên CSDL mà không cần di chuyển dữ liệu. | **SQL Generation:** Tự động sinh ra các câu lệnh `SELECT COUNT... WHERE...` để CSDL tự thực hiện việc kiểm tra, giảm tải cho ứng dụng. |
| *"Mô hình dữ liệu không có tài liệu, phải dò dẫm từng mối quan hệ."* | **Khám phá Mối quan hệ Tự động:** Tự động vẽ lại bản đồ schema dựa trên cả khóa ngoại và suy luận thông minh. | **`RelationshipAnalyzer`:** Áp dụng thuật toán Hybrid, kết hợp `Foreign Keys` và các quy ước đặt tên (`user_id` -> `users.id`) với thư viện `inflect`. |
| *"Phát hiện vấn đề xong không biết phải làm gì tiếp theo."* | **Kế hoạch Hành động & Công cụ Làm sạch:** Tự động tạo ra danh sách các bước cần làm và cung cấp công cụ để trích xuất dữ liệu sạch chỉ với một cú nhấp chuột. | **`ProfilingAdapter` & `DQ Center`:** Logic "làm giàu" kết quả và giao diện cho phép người dùng chọn quy tắc vi phạm để lọc và tải về file CSV đã được làm sạch. |

---

### **3. Bản thiết kế Chiến lược: Tại sao là Clean Architecture?**

Lựa chọn Clean Architecture không phải là một quyết định kỹ thuật, đó là một **quyết định chiến lược** về khả năng phát triển bền vững. Nó đảm bảo DA-ODA sẽ không trở thành một "mớ bòng bong" khi quy mô lớn hơn.


* **Lõi (Core Engine):** Trái tim hiệu năng cao, chỉ làm việc với Polars, hoàn toàn độc lập.
* **Dịch vụ (Service Layer):** Bộ não điều phối, chứa logic nghiệp vụ, tuân thủ triệt để Dependency Injection.
* **Ngoại vi & CSDL (Periphery & Database):** Cánh tay và bộ nhớ, giao tiếp với thế giới bên ngoài (Connectors) và lưu trữ metadata (SQLAlchemy + Alembic).
* **Giao diện (UI Layer):** Bộ mặt tinh gọn, chỉ có nhiệm vụ hiển thị dữ liệu đã được xử lý sẵn.

Kiến trúc này đảm bảo ngày mai, việc thay thế `PostgreSQL` bằng `Snowflake`, hay `Streamlit` bằng một `REST API` sẽ không làm phá vỡ toàn bộ hệ thống.

---

### **4. Case Study: Minh chứng Năng lực Giải quyết Vấn đề Chuyên sâu**

#### **Case Study 1: `DataSampler` - Nghệ thuật Lấy mẫu Tối ưu**
Module này cho thấy khả năng vượt ra ngoài các giải pháp có sẵn để giải quyết một bài toán chuyên sâu. Thay vì một cách tiếp cận "one-size-fits-all", nó triển khai:
* **Chiến lược Đa dạng:** Tự động nhận diện `PostgreSQL`, `MySQL`... để chọn phương án tối ưu nhất.
* **Hiệu năng I/O là Vua:** Quyết định sử dụng `ConnectorX` (viết bằng Rust) thay vì `pandas.read_sql` là một lựa chọn có chủ đích để tối đa hóa tốc độ đọc dữ liệu.
* **Metadata-Driven:** Nó thu thập metadata (kích thước, cột ngày tháng...) *trước khi* lấy mẫu, giúp người dùng ra quyết định thông minh hơn.

#### **Case Study 2: `RelationshipAnalyzer` - Trí tuệ Suy luận trong Khám phá Schema**
Đây không phải là một công cụ đọc metadata đơn thuần. Nó hành động như một nhà phân tích có kinh nghiệm:
* **Cách tiếp cận Hybrid:** Kết hợp nguồn chân lý (khóa ngoại đã định nghĩa) và suy luận dựa trên quy ước (`table_singular_id` -> `table_plural.id`).
* **Sự tinh tế trong xử lý:** Việc sử dụng thư viện `inflect` để xử lý tên bảng số ít/số nhiều (user vs. users) là một chi tiết nhỏ nhưng thể hiện sự tỉ mỉ và giải quyết một vấn đề rất thực tế.

---

### **5. Kết luận: Một Tuyên ngôn về Năng lực**

**DA-ODA** không chỉ là một dự án trong portfolio. **Đó là một tuyên ngôn** về khả năng:
* **Tư duy Hệ thống:** Áp dụng thành thạo các nguyên tắc kiến trúc phần mềm hiện đại (Clean Architecture, DI, SOLID) vào một bài toán dữ liệu thực tế.
* **Kiến thức Chuyên sâu:** Am hiểu sâu sắc về hiệu năng của các công cụ (Polars vs. Pandas), các hệ CSDL khác nhau, và các thuật toán chuyên biệt.
* **Giải quyết Vấn đề Thực chiến:** Tập trung vào việc giải quyết những "nỗi đau" thực sự của người dùng cuối, từ đó tạo ra một sản phẩm không chỉ mạnh về kỹ thuật mà còn có giá trị sử dụng cao.

Đây là minh chứng cho một hệ thống phần mềm được thiết kế có chủ đích, hiệu quả và bền vững.