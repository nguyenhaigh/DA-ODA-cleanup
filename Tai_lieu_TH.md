
-----

# **Tài liệu Kỹ thuật Tổng hợp & Bằng chứng Triển khai: Dự án DA-ODA**

### **Giới thiệu**

Tài liệu này cung cấp một cái nhìn sâu vào chi tiết kỹ thuật và triển khai của dự án **DA-ODA**, đóng vai trò là bằng chứng củng cố cho các luận điểm đã nêu trong **"Hồ sơ Năng lực Kỹ thuật & Kiến trúc Dự án"**. Tài liệu sẽ đi qua từng lớp kiến trúc, phân tích các thành phần cốt lõi và làm rõ cách chúng tương tác với nhau để hiện thực hóa tầm nhìn của dự án: biến quy trình phân tích dữ liệu thành một trải nghiệm **tốc độ, tự động và đáng tin cậy**.

### **Kiến trúc Tổng thể - Hiện thực hóa Clean Architecture**

Như đã nêu trong hồ sơ năng lực, DA-ODA tuân thủ nghiêm ngặt triết lý **Clean Architecture**. Sự phân tách thành các lớp độc lập được thể hiện rõ qua cấu trúc thư mục và sự phụ thuộc giữa các module. Dưới đây là phân tích chi tiết từng lớp.

-----

<br>

## **Phần 1: Lớp 1 - Lõi Phân Tích (Core Engine)**

Đây là "vùng đất của hiệu năng và logic thuần túy", hoàn toàn độc lập với các chi tiết bên ngoài.

### **1.1. `Profiler Engine` (`profiler.py`)**

  * **Trách nhiệm:** Thực thi profiling toàn diện trên một Polars DataFrame, tính toán các chỉ số thống kê, và tự động phát hiện các "insights" (gợi ý) về chất lượng dữ liệu.
  * **Minh chứng độc lập:** Module này chỉ `import polars` và các thư viện chuẩn, không hề có `sqlalchemy` hay `streamlit`, xác nhận nó không biết gì về nguồn dữ liệu hay cách hiển thị.
  * **Bằng chứng hiệu năng & logic:** Tự động phát hiện các vấn đề như suy luận kiểu dữ liệu, giá trị âm bất thường, kiểm tra viết hoa không nhất quán, và phân tích các giá trị ngoại lai.

### **1.2. `Data Quality Engine` (`data_quality_engine.py`)**

  * **Trách nhiệm:** Cung cấp một framework mạnh mẽ để thực thi các quy tắc chất lượng dữ liệu.
  * **Minh chứng kiến trúc:** Sử dụng **Mô hình Thực thi Hỗn hợp (Hybrid Execution Model)**:
      * **Thực thi trên CSDL:** Sinh ra các câu lệnh SQL để CSDL tự thực thi việc đếm lỗi cho các quy tắc như `not_null`, `unique`, `foreign_key`.
      * **Thực thi trên Metadata:** Chạy kiểm tra trên kết quả profiling đã có cho các quy tắc `statistical_anomaly`.

### **1.3. `Correlation Analyzer` (`analyzer.py`)**

  * **Trách nhiệm:** Tính toán ma trận tương quan Pearson hiệu năng cao.
  * **Minh chứng bền bỉ:** Có khả năng xử lý các trường hợp đặc biệt như kiểu dữ liệu `Decimal` và các giá trị `null` một cách tự động.

### **1.4. `Relationship Analyzer` (`relationship_analyzer.py`)**

  * **Trách nhiệm:** Tự động khám phá các mối quan hệ khóa ngoại.
  * **Minh chứng trí tuệ suy luận:** Áp dụng cách tiếp cận Hybrid thông minh:
    1.  **Ưu tiên hàng đầu:** Đọc các khóa ngoại đã được định nghĩa tường minh.
    2.  **Suy luận theo quy ước:** Áp dụng các quy tắc dựa trên quy ước đặt tên (ví dụ: `user_id` -\> bảng `users`.`id`), sử dụng thư viện `inflect` để xử lý tên số ít/số nhiều.

-----

<br>

## **Phần 2: Lớp 2 - Dịch vụ & Điều phối (Service Layer)**

Đây là "bộ não" điều phối, nơi các logic nghiệp vụ được kết hợp lại. Lớp này tuân thủ chặt chẽ nguyên tắc **Dependency Injection (DI)**.

### **2.1. `ProfilingService` (`profiling_service.py`)**

  * **Trách nhiệm:** Là **"nhạc trưởng"** điều phối toàn bộ quy trình profiling cho một bảng trong CSDL.
  * **Minh chứng DI:** Hàm `__init__` nhận vào các đối tượng `connector`, `governance_service`, `comparison_service`, `connection_service`, `repo`, thể hiện rõ việc các dependency được "tiêm vào" từ bên ngoài.
  * **Luồng điều phối (`profile_table`):** Hiện thực hóa luồng "Từ Click đến Insight" bằng cách gọi tuần tự các thành phần khác: `DataSampler` -\> `Profiler Engine` -\> `MetadataRepository` -\> `DataGovernanceService` -\> `ProfilingAdapter`.

### **2.2. `DataGovernanceService` (`governance_service.py`)**

  * **Trách nhiệm:** Trung tâm điều phối tất cả các tác vụ quản trị dữ liệu.
  * **Minh chứng điều phối:** Cung cấp các phương thức để quản lý (CRUD) các quy tắc DQ và điều phối việc thực thi kiểm tra bằng cách gọi `DataQualityEngine`, sau đó lưu kết quả qua `MetadataRepository`.

### **2.3. `ConnectionService` (`connection_service.py`)**

  * **Trách nhiệm:** Quản lý toàn bộ vòng đời của các kết nối đến nguồn dữ liệu.
  * **Minh chứng kiến trúc mở rộng:** Sử dụng kiến trúc **Connector Factory** (`CONNECTOR_MAP`) để dễ dàng thêm các nguồn dữ liệu mới. Tích hợp `CryptoService` để mã hóa/giải mã mật khẩu một cách an toàn.

-----

<br>

## **Phần 3: Lớp 3 - Giao tiếp & Lưu trữ (Periphery & Persistence)**

Đây là lớp ngoài cùng, chịu trách nhiệm cho mọi hoạt động I/O.

### **3.1. Hệ thống Connectors (`src/periphery/connectors/`)**

  * **`BaseConnector`:** Một "bản hợp đồng" (Lớp cơ sở trừu tượng) định nghĩa các hành vi mà mọi connector phải có.
  * **Các Connector Cụ thể:** `PostgresConnector` và `MySQLConnector` cung cấp logic kết nối và các câu lệnh SQL đặc thù cho từng hệ CSDL.
  * **`FileConnector`:** Một connector chuyên biệt "giả lập" nguồn dữ liệu từ file, cho phép hệ thống làm việc với file một cách thống nhất.

### **3.2. Lớp Lưu trữ Metadata (`src/database/`)**

  * **Mô hình Dữ liệu (`models.py`):** "Bản thiết kế" schema cho CSDL metadata của ứng dụng, được định nghĩa bằng SQLAlchemy ORM, bao gồm các bảng như `Connection`, `ProfilingRun`, `DataQualityRule`, v.v. Các mối quan hệ và hành vi xóa theo tầng (`cascade`) được định nghĩa rõ ràng.
  * **Repository (`repository.py`):** Tuân thủ **Mô hình Repository**, đóng vai trò là **cổng giao tiếp duy nhất** tới CSDL metadata. Nó đóng gói toàn bộ logic tương tác với SQLAlchemy, cung cấp các phương thức nghiệp vụ rõ ràng cho lớp Service.

-----

<br>

## **Phần 4: Lớp 4 - Giao diện Người dùng (UI Layer)**

Được xây dựng bằng **Streamlit**, UI là "bộ mặt" của DA-ODA, cung cấp trải nghiệm tương tác để khai thác sức mạnh của hệ thống backend.

### **4.1. Trang Profiler & Phân tích**

  * **`Profiler`:** Trang chính của ứng dụng, nơi người dùng kết nối, chọn bảng, cấu hình lấy mẫu và chạy phân tích. Kết quả hiển thị bao gồm **Bảng điểm Sức khỏe**, **Kế hoạch Hành động**, và **Phân tích chi tiết** từng cột với các biểu đồ tương tác.
  * **`Correlation Analysis` & `Relationship Explorer`:** Các trang phụ trợ cho phép người dùng đào sâu vào phân tích tương quan và khám phá mối quan hệ schema dưới dạng biểu đồ ERD.

### **4.2. Quản trị & So sánh**

  * **`Data Quality Center`:** Một trung tâm chỉ huy cho phép quản lý (thêm/xem/xóa) các quy tắc DQ, chạy kiểm tra, xem chi tiết các dòng lỗi, và đặc biệt là cung cấp công cụ **"Làm sạch & Trích xuất"** để tải về dữ liệu đã được lọc bỏ các dòng lỗi.
  * **`Snapshot Management & Comparison`:** Các trang cho phép quản lý các "Golden Snapshots" và thực hiện so sánh sâu giữa hai snapshot bất kỳ, làm nổi bật các thay đổi về cấu trúc và chỉ số.

-----

<br>

## **Kết luận**

Tài liệu này đã cung cấp các bằng chứng kỹ thuật chi tiết, xác nhận rằng **DA-ODA** không chỉ là một tập hợp các script mà là một hệ thống phần mềm được thiết kế có chủ đích. Việc áp dụng nhất quán các nguyên tắc kiến trúc hiện đại như Clean Architecture, Dependency Injection, và Repository Pattern, kết hợp với kiến thức chuyên sâu về hệ sinh thái dữ liệu (Polars, ConnectorX, SQLAlchemy), đã tạo nên một nền tảng vững chắc, hiệu quả và sẵn sàng cho việc phát triển bền vững trong tương lai.