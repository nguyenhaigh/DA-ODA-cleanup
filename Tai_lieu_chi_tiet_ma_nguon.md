
---

# **Tài liệu Kỹ thuật DA-ODA **

## **Phần 1: Lõi Phân Tích (Core Engine) - Trái Tim Của Hệ Thống**

Đây là khu vực hiệu suất cao, chịu trách nhiệm cho tất cả các phép tính toán và phân tích dữ liệu nặng. Logic trong `src/core/` được thiết kế để hoàn toàn độc lập với giao diện người dùng (UI) và lớp ứng dụng (Service), chỉ làm việc trực tiếp với Polars DataFrame hoặc thực thi các truy vấn SQL tối ưu để đảm bảo tốc độ tối đa.

---

### **1.1. `Profiler Engine` (`src/core/engine/profiler.py`)**

**Vai trò cốt lõi:** Thực thi một quy trình profiling toàn diện trên một Polars DataFrame, tính toán các chỉ số thống kê, phát hiện các insight ngữ cảnh và tạo ra dữ liệu nền tảng cho việc trực quan hóa.

**Nguyên tắc thiết kế:**
* **Chính xác (Accurate):** Mọi công thức thống kê đều tuân thủ các định nghĩa chuẩn, đảm bảo tính đúng đắn của kết quả.
* **Hiệu năng (Performant):** Tận dụng tối đa khả năng xử lý song song và các biểu thức (expressions) của Polars để giảm thiểu số lần quét dữ liệu. Các cột có kiểu `Decimal` được tự động ép sang `Float64` để tương thích và tăng tốc các phép tính.
* **Bền bỉ (Robust):** Sử dụng các cơ chế xử lý lỗi cục bộ (`_execute_single_expression`) để một lỗi tính toán trên một chỉ số nhỏ không làm sập toàn bộ quy trình profiling.
* **Linh hoạt (Flexible):** Cho phép ghi đè các tham số quan trọng từ bên ngoài (ví dụ: số `bins` của histogram) để tùy chỉnh độ chi tiết của phân tích.

**Các Chỉ số & Tính năng Chính:**

#### A. Chỉ số Toàn cục (Table-level Metrics)
* `full_row_duplicates`: Đếm số dòng bị trùng lặp hoàn toàn, có logic thông minh loại trừ các cột khóa chính (`pk_columns`) và các cột văn bản tự do (`notes`, `description`) để có kết quả chính xác hơn.

#### B. Chỉ số Cấp độ Cột (Column-level Metrics)

| Loại Dữ liệu | Các Chỉ số Chính |
| :--- | :--- |
| **Chung** | `total_count`, `null_count` (% null), `distinct_count`, `duplicate_count` |
| **Số (Numeric)** | `mean`, `stddev`, `min`, `max`, `median`, `p25`, `p75`, `zero_count`, `skewness` (độ lệch), dữ liệu **histogram** (bao gồm cả dữ liệu đã chuẩn hóa), và hình ảnh **sparkline** SVG. |
| **Chuỗi (String)** | `blank_count`, `avg_length`, `min_length`, `max_length`, và danh sách các giá trị phổ biến nhất (`frequent_values`). |
| **Ngày giờ (Datetime)** | `min_date`, `max_date`. |
| **Logic (Boolean)**| `true_count` (% true), `false_count` (% false). |

#### C. Hệ thống "Insights" Tự động

Đây là tính năng vượt trội của `Profiler Engine`, tự động phát hiện các vấn đề tiềm ẩn hoặc các cơ hội làm sạch dữ liệu:

* **Suy luận Kiểu dữ liệu (`_analyze_inferred_types`):** Phân tích các cột chuỗi để đề xuất chuyển đổi sang kiểu `Numeric` hoặc `Datetime` nếu tỷ lệ chuyển đổi thành công vượt ngưỡng (`inferred_type_threshold`).
* **Phát hiện Viết hoa Không nhất quán (`_analyze_casing_consistency`):** Tìm thấy các biến thể của cùng một giá trị do viết hoa khác nhau (ví dụ: "Hà Nội" và "hà nội").
* **Cảnh báo Ngữ cảnh (Contextual Warnings):**
    * `NEGATIVE_VALUES_DETECTED`: Cảnh báo khi phát hiện giá trị âm trong các cột thường là số dương (ví dụ: `age`, `quantity`, `id`).
    * `OUT_OF_RANGE`: Cảnh báo khi giá trị nằm ngoài một khoảng logic đã biết (ví dụ: `rating > 5`).
    * `FUTURE_DATE_DETECTED`: Cảnh báo khi phát hiện ngày tháng nằm trong tương lai.
    * `TRAILING_WHITESPACE`: Phát hiện các giá trị có khoảng trắng thừa ở đầu/cuối.
* **Phân tích Ngoại lai (`_calculate_outliers_iqr`):** Sử dụng phương pháp IQR (Interquartile Range) để xác định, đếm số lượng và cung cấp mẫu các giá trị ngoại lai. Đồng thời, engine này trả về đầy đủ các thông số cần thiết (`q1`, `median`, `q3`, `whisker_low`, `whisker_high`) để vẽ biểu đồ hộp (box plot).

---

### **1.2. `Data Quality Engine` (`src/core/data_quality_engine.py`)**

**Vai trò cốt lõi:** Cung cấp một framework mạnh mẽ để định nghĩa, thực thi và lấy kết quả chi tiết từ các quy tắc kiểm tra chất lượng dữ liệu.

**Kiến trúc & Triết lý thiết kế:**
* **Hybrid Execution Model:** Đây là điểm kiến trúc quan trọng nhất. Engine có khả năng:
    1.  **Thực thi trên CSDL (`On-Database`):** Đối với các quy tắc cấp độ dòng (`not_null`, `unique`, `foreign_key`...), engine sẽ sinh ra các câu lệnh SQL để CSDL tự thực thi việc đếm lỗi. Cách này cực kỳ hiệu quả vì không cần tải dữ liệu về ứng dụng.
    2.  **Thực thi trên Metadata (`On-Profile-Results`):** Đối với quy tắc `statistical_anomaly`, engine sẽ thực thi logic trực tiếp trên kết quả do `Profiler Engine` tạo ra, cho phép kiểm tra các chỉ số tổng hợp (ví dụ: `mean`, `stddev`, `null_percentage`).
* **Mở rộng qua Dispatcher:** Sử dụng "bảng điều phối" (`_db_check_handlers`) để ánh xạ tên quy tắc tới hàm xử lý tương ứng, giúp việc thêm quy tắc mới trở nên an toàn và dễ dàng.

**Các Chức năng Chính:**

* **`run_check`:** Là hàm điều phối chính, nhận vào một định nghĩa quy tắc, quyết định xem nên chạy trên CSDL hay trên kết quả profiling, và trả về một kết quả chuẩn hóa bao gồm: `status` (pass/fail/error), `failing_rows_count`, và `details`.
* **`get_failing_rows_for_check`:** Cung cấp khả năng truy vấn và lấy về **dữ liệu thực tế của các dòng bị lỗi** cho một quy tắc cụ thể, giúp người dùng chẩn đoán vấn đề tận gốc.
* **`get_clean_data`:** Tự động xây dựng và thực thi một câu lệnh SQL duy nhất để trích xuất một tập dữ liệu "sạch", đã được lọc bỏ tất cả các dòng vi phạm một bộ quy tắc cho trước.

**Các loại Quy tắc được Hỗ trợ:**

| Tên Quy tắc | Mô tả | Phương thức Thực thi |
| :--- | :--- | :--- |
| `not_null` | Kiểm tra giá trị rỗng. | Trên CSDL |
| `unique` | Kiểm tra giá trị trùng lặp. | Trên CSDL |
| `accepted_values` | Kiểm tra giá trị có nằm trong một danh sách cho phép không. | Trên CSDL |
| `cross_column_logic`| Thực thi một biểu thức SQL tùy chỉnh để kiểm tra logic giữa các cột. | Trên CSDL |
| `foreign_key` | Kiểm tra tính toàn vẹn tham chiếu (không có khóa ngoại mồ côi). | Trên CSDL |
| `statistical_anomaly`| So sánh một chỉ số thống kê từ profiler (ví dụ: `mean`, `max`, `null_percentage`) với một ngưỡng cho trước. | Trên Kết quả Profiling |

---

### **1.3. `Correlation Analyzer` (`src/core/engine/analyzer.py`)**

**Vai trò cốt lõi:** Tính toán ma trận tương quan Pearson giữa các cột số trong một DataFrame, giúp khám phá mối quan hệ tuyến tính giữa các biến.

**Kiến trúc & Triết lý thiết kế:**
* **Bền bỉ:** Tự động xử lý các trường hợp đặc biệt có thể gây lỗi. Cụ thể, engine sẽ tự động ép kiểu các cột `Decimal` sang `Float64` và loại bỏ tất cả các dòng có chứa giá trị `null` trên các cột đang xét trước khi tính toán.
* **Tập trung:** Chỉ thực hiện một nhiệm vụ duy nhất là tính toán tương quan và trả về kết quả có cấu trúc.

**Luồng hoạt động (Workflow):**
1.  **Đầu vào:** Nhận một Polars DataFrame và danh sách các cột số cần phân tích.
2.  **Làm sạch:** Tạo một DataFrame tạm thời, chỉ chứa các cột đã chọn và đã được làm sạch (ép kiểu Decimal, loại bỏ `null`).
3.  **Tính toán:** Gọi hàm `corr()` hiệu năng cao của Polars để tính toán ma trận tương quan.
4.  **Phân tích kết quả:** Lặp qua ma trận để xác định các cặp cột có hệ số tương quan (giá trị tuyệt đối) vượt một ngưỡng đã định nghĩa (`strong_correlation_threshold`).
5.  **Đầu ra:** Trả về một đối tượng JSON chứa:
    * `correlation_matrix`: Toàn bộ ma trận tương quan.
    * `strong_correlations_raw`: Một danh sách chỉ chứa các cặp tương quan mạnh.
    * `metadata`: Các thông tin ngữ cảnh về quá trình phân tích.

---

### **1.4. `Relationship Analyzer` (`src/core/relationship_analyzer.py`)**

**Vai trò cốt lõi:** Tự động khám phá các mối quan hệ khóa ngoại tiềm năng trong một schema CSDL bằng cách kết hợp metadata chính thức và các quy tắc suy nghiệm dựa trên quy ước đặt tên.

**Kiến trúc & Triết lý thiết kế:**
* **Ưu tiên sự Chính xác (Defined-First Approach):** Thuật toán luôn ưu tiên các mối quan hệ đã được định nghĩa tường minh trong CSDL (`FOREIGN KEY constraints`). Đây là nguồn thông tin đáng tin cậy nhất.
* **Suy luận Thông minh (Convention-based Heuristics):** Đối với các mối quan hệ chưa được định nghĩa, engine áp dụng một chuỗi các quy tắc suy luận dựa trên kinh nghiệm thực tế về thiết kế CSDL.
* **Hiệu quả:** Tải và cache lại metadata của schema (`_load_all_schemas`) để tránh các lệnh gọi lặp đi lặp lại tới CSDL.

**Thuật toán Khám phá:**

1.  **Bước 1: Tìm các Khóa ngoại Tường minh (`_get_defined_relationships`)**
    * Sử dụng `inspector.get_foreign_keys()` để lấy tất cả các mối quan hệ đã được định nghĩa. Các mối quan hệ này được coi là "sự thật" và được thêm vào kết quả ngay lập tức.

2.  **Bước 2: Xây dựng Bản đồ Khóa chính (`_get_potential_primary_keys`)**
    * Lấy tất cả các khóa chính (`PRIMARY KEY`) từ tất cả các bảng để làm cơ sở cho việc đối chiếu ở bước sau.

3.  **Bước 3: Áp dụng Quy ước Đặt tên (`discover_relationships_by_convention`)**
    * Engine lặp qua từng cột trong từng bảng (trừ các cột đã có FK và các cột là PK của chính bảng đó) và áp dụng các quy tắc theo thứ tự ưu tiên:
        * **Quy ước 1 (Ưu tiên cao nhất): `table_singular_id` -> `table_plural.id`**
            * Phát hiện cột có tên dạng `<tên_bảng_đích_số_ít>_id` (ví dụ: `user_id` trong bảng `orders`).
            * Sử dụng thư viện `inflect` để chuyển `user` thành `users`.
            * Kiểm tra xem bảng `users` có tồn tại và có khóa chính là `id` hay không. Nếu có, một mối quan hệ được đề xuất.
        * **Quy ước 2: `column_name` -> `other_table.column_name` (PK)**
            * Phát hiện một cột (ví dụ: `customer_key`) có tên trùng khớp với tên của một khóa chính ở một bảng khác.

---

## **Phần 2: Lớp Dịch vụ & Điều phối (Service & Orchestration Layer)**

Đây là "bộ não" của ứng dụng DA-ODA, nơi chứa đựng toàn bộ logic nghiệp vụ. Lớp Service đóng vai trò trung gian, điều phối hoạt động giữa Giao diện người dùng (UI), Lõi xử lý (Core Engine), và lớp Truy cập Dữ liệu (Database Repository). Các service được thiết kế theo nguyên tắc **Dependency Injection**, nhận các dependency cần thiết (như Repository, Connector, các service khác) khi khởi tạo, giúp hệ thống trở nên module hóa, dễ kiểm thử và bảo trì.

---

### **2.1. `ProfilingService` (`src/services/profiling_service.py`)**

**Vai trò cốt lõi:** Là **"nhạc trưởng"** điều phối toàn bộ quy trình profiling cho một bảng trong **cơ sở dữ liệu**. Service này chịu trách nhiệm thực hiện một chuỗi các tác vụ phức tạp theo đúng thứ tự để tạo ra một "Báo cáo Sức khỏe Dữ liệu" (Data Health Report) hoàn chỉnh.

**Luồng nghiệp vụ chính - `profile_table()`:**
Đây là phương thức trung tâm, thực hiện một chuỗi 7 bước tuần tự và chặt chẽ:
1.  **Lấy Dữ liệu Mẫu (`_sample_data_for_profiling`):** Gọi `DataSampler` từ Core Engine để lấy một DataFrame mẫu từ CSDL dựa trên chiến lược do người dùng lựa chọn (Full Load, Adaptive, Time-based).
2.  **Thực thi Profiling Cốt lõi (`_execute_core_profiling`):** Chuyển DataFrame mẫu vào `profile_dataframe` (từ Core Engine) để thực hiện các phép tính thống kê nặng. Kết quả trả về là một dictionary chứa cả `column_profiles` và `table_metrics`.
3.  **Tổng hợp Dữ liệu Sơ bộ:** Tạo một cấu trúc dữ liệu JSON hoàn chỉnh, kết hợp thông tin về bảng (`table_info`), các chỉ số của bảng (`table_metrics`) và danh sách profile của các cột (`columns`). Cấu trúc này sẽ được sử dụng cho các bước tiếp theo.
4.  **Lưu Kết quả vào CSDL (`_save_run_to_database`):** Gửi đối tượng JSON ở trên đến `MetadataRepository` để lưu lại. CSDL sẽ trả về một đối tượng `ProfilingRun` chứa `run_id` duy nhất cho lần chạy này.
5.  **Chạy Kiểm tra Chất lượng Dữ liệu (`_run_data_quality_checks`):** Với `run_id` vừa có, service này gọi `DataGovernanceService` để thực thi tất cả các quy tắc DQ đang hoạt động cho bảng. **Quan trọng:** Nó cũng truyền vào kết quả profiling vừa tính toán ở bước 2, giúp các quy tắc `statistical_anomaly` có thể chạy ngay mà không cần truy vấn lại CSDL.
6.  **Tổng hợp và Hoàn thiện (`_adapt_and_finalize_results`):** Tất cả các "mảnh ghép" (kết quả profiling thô, kết quả DQ, DataFrame mẫu) được đưa vào `ProfilingAdapter`. Adapter sẽ làm giàu dữ liệu, tính toán điểm sức khỏe (health score), và tạo ra cấu trúc JSON cuối cùng để hiển thị trên UI.
7.  **Lấy Thông tin Baseline (`_get_baseline_info_for_ui`):** Truy vấn `MetadataRepository` để tìm `run_id` của lần chạy lịch sử gần nhất, giúp UI có thể hiển thị tùy chọn so sánh.

**Các chức năng quan trọng khác:**
* `get_sampling_prerequisites`: Thu thập các metadata cần thiết (số dòng, kích thước, cột datetime) để UI có thể hiển thị các tùy chọn lấy mẫu phù hợp.
* `pin_from_snapshot` & `delete_golden_snapshot`: Quản lý vòng đời của các "Golden Snapshots" bằng cách ghi hoặc xóa các file JSON tương ứng.
* `analyze_correlation`: Điều phối việc phân tích tương quan bằng cách gọi `CorrelationAnalyzer` từ Core Engine.

---

### **2.2. `FileProfilingService` (`src/services/file_profiling_service.py`)**

**Vai trò cốt lõi:** Là một phiên bản **chuyên biệt và gọn nhẹ** của `ProfilingService`, được thiết kế để xử lý duy nhất nguồn dữ liệu từ file (CSV, Excel...).

**Kiến trúc & Sự khác biệt chính:**
* **Kế thừa và Ghi đè:** Kế thừa từ `ProfilingService` nhưng ghi đè (override) phương thức `profile_table` với một luồng xử lý đơn giản hơn rất nhiều.
* **Không Tương tác CSDL Metadata:** Đây là điểm khác biệt lớn nhất. Quy trình profiling file hoàn toàn không lưu bất kỳ thông tin nào vào CSDL metadata. Nó không tạo `run_id`, không lưu kết quả, và do đó, **không chạy các quy tắc Data Quality**.

**Luồng nghiệp vụ `profile_table()`:**
1.  **Lấy Dữ liệu:** Gọi `FileConnector` để đọc toàn bộ file vào một Polars DataFrame.
2.  **Thực thi Profiling Cốt lõi:** Gọi trực tiếp hàm `profile_dataframe` từ Core Engine.
3.  **Tổng hợp qua Adapter:** Đưa kết quả thô vào `ProfilingAdapter` để làm giàu và định dạng.
4.  **Trả về Kết quả:** Trả về một dictionary chứa kết quả cuối cùng, sẵn sàng cho UI hiển thị.

---

### **2.3. `DataGovernanceService` (`src/services/governance_service.py`)**

**Vai trò cốt lõi:** Là trung tâm điều phối tất cả các tác vụ liên quan đến quản trị dữ liệu, bao gồm quản lý quy tắc DQ và khám phá mối quan hệ schema.

**Các Nhóm Chức năng Chính:**

#### A. Quản lý & Thực thi Quy tắc DQ:
* **Quản lý (CRUD):** Cung cấp các phương thức để UI có thể tạo (`create_dq_rule`), đọc (`get_active_rules_for_table`), và xóa (`delete_dq_rule`) các quy tắc DQ. Các tác vụ này được ủy quyền xuống cho `MetadataRepository`.
* **Thực thi (`run_all_dq_checks_for_table`):**
    * Lấy tất cả các quy tắc đang hoạt động cho bảng từ Repository.
    * **Logic thông minh:** Kiểm tra xem có quy tắc loại `statistical_anomaly` hay không. Nếu có, nó sẽ chủ động tải dữ liệu profiling từ CSDL (dựa trên `run_id`) nếu dữ liệu này không được `ProfilingService` cung cấp sẵn.
    * Lặp qua từng quy tắc và gọi `dq_engine.run_check` để thực thi.
    * Lưu từng kết quả pass/fail vào CSDL qua Repository.
* **Truy vấn Dữ liệu Lỗi (`get_failing_rows`, `get_clean_data_as_df`):** Đóng vai trò cầu nối, nhận yêu cầu từ UI và gọi các phương thức tương ứng của `DataQualityEngine` để lấy về các dòng dữ liệu bị lỗi hoặc đã được làm sạch.

#### B. Khám phá Mối quan hệ:
* `discover_and_save_relationships`: Kích hoạt `RelationshipAnalyzer` từ Core Engine để phân tích schema, sau đó nhận kết quả và lưu chúng vào CSDL thông qua Repository.
* `get_all_relationships`: Lấy danh sách các mối quan hệ đã được khám phá từ Repository để hiển thị trên UI.

---

### **2.4. `ConnectionService` (`src/services/connection_service.py`)**

**Vai trò cốt lõi:** Là service duy nhất trong hệ thống chịu trách nhiệm **tạo, lưu trữ, kích hoạt, và quản lý** tất cả các kết nối đến nguồn dữ liệu.

**Kiến trúc & Cơ chế hoạt động:**
* **Connector Factory (`CONNECTOR_MAP`):** Sử dụng một dictionary `CONNECTOR_MAP` để ánh xạ một chuỗi `db_type` (ví dụ: "postgres") tới lớp `Connector` và lớp `Pydantic Params` tương ứng. Kiến trúc này giúp dễ dàng thêm các nguồn dữ liệu mới mà không cần sửa đổi nhiều code.
* **Active Connector:** Tại một thời điểm, service chỉ quản lý một kết nối đang hoạt động (`active_connector`). Các service khác (như `ProfilingService`) sẽ lấy connector này để làm việc.
* **Tích hợp CryptoService:** Tự động gọi `CryptoService` để mã hóa mật khẩu trước khi lưu vào CSDL và giải mã sau khi tải lên, đảm bảo an toàn cho thông tin nhạy cảm.

**Luồng nghiệp vụ quan trọng:**
* **Lưu kết nối (`save_connection`):** Nhận thông tin kết nối, mã hóa mật khẩu, và ủy quyền cho `MetadataRepository` lưu vào CSDL.
* **Kích hoạt kết nối đã lưu (`activate_connection_by_id`):**
    1.  Lấy thông tin kết nối từ Repository dựa trên `id`.
    2.  Giải mã mật khẩu.
    3.  Sử dụng `CONNECTOR_MAP` để xác định và khởi tạo đúng lớp Connector.
    4.  Gọi `connector.test_connection()` để xác thực.
    5.  Nếu thành công, đặt connector này làm `active_connector`.

---

### **2.5. `ComparisonService` (`src/services/comparison_service.py`)**

**Vai trò cốt lõi:** Thực hiện một phép so sánh sâu (deep comparison) giữa hai bản ghi profiling (snapshots) để xác định mọi sự khác biệt.

**Cơ chế hoạt động:**
* **Hàm Đệ quy (`_find_diffs`):** Cốt lõi của service là một hàm đệ quy có khả năng duyệt qua hai cấu trúc dictionary/JSON phức tạp. Nó có thể phát hiện:
    * Sự thay đổi giá trị của một trường.
    * Các trường/cột được thêm mới.
    * Các trường/cột bị xóa đi.
* **Luồng so sánh chính (`compare_profiling_snapshots_detailed`):**
    1.  Nhận vào hai `run_id`.
    2.  Tải toàn bộ dữ liệu JSON của hai snapshot từ `MetadataRepository`.
    3.  Gọi hàm `_find_diffs` trên từng phần chính của snapshot: `table_info`, `health_report`, và từng cột riêng lẻ.
    4.  Tổng hợp tất cả các khác biệt tìm thấy vào một cấu trúc kết quả duy nhất, trả về cho UI để hiển thị.

---

### **2.6. `CryptoService` (`src/services/crypto_service.py`)**

**Vai trò cốt lõi:** Một service tiện ích đơn giản, tập trung cung cấp chức năng mã hóa và giải mã đối xứng, chủ yếu được `ConnectionService` sử dụng để bảo vệ mật khẩu CSDL.

**Cơ chế hoạt động:**
* Sử dụng thư viện `cryptography.fernet` để thực hiện mã hóa.
* **Bảo mật:** Chìa khóa mã hóa không được lưu trữ trong mã nguồn. Thay vào đó, nó được đọc từ biến môi trường `ENCRYPTION_KEY` (thông qua file `.env`), tuân thủ các nguyên tắc bảo mật tốt nhất.
Chắc chắn rồi. Giờ đây với mã nguồn đầy đủ của `repository.py` và `models.py`, tôi sẽ viết lại tài liệu cho **Nhóm 3** một cách hoàn toàn chính xác, không còn các giả định. Tài liệu này sẽ phản ánh đúng cấu trúc CSDL metadata và cách lớp Repository tương tác với nó.

---

## **Phần 3: Lớp Giao tiếp & Lưu trữ (Periphery & Persistence Layer)**

Đây là lớp ngoài cùng trong kiến trúc của DA-ODA, chịu trách nhiệm cho mọi hoạt động I/O (Input/Output). Nó đóng vai trò là "cánh tay" và "bộ nhớ dài hạn" của ứng dụng, xử lý hai nhiệm vụ chính:
1.  **Giao tiếp với Nguồn dữ liệu ngoài (External Data Sources):** Thông qua một hệ thống **Connector** linh hoạt và có khả năng mở rộng.
2.  **Lưu trữ Trạng thái & Metadata (`Persistence`):** Lưu trữ toàn bộ thông tin về các kết nối đã lưu, các lần chạy profiling, quy tắc DQ, v.v., vào một cơ sở dữ liệu riêng của ứng dụng.

---

### **3.1. Hệ thống Connectors (`src/periphery/connectors/`)**

**Triết lý thiết kế:**
Hệ thống Connector được xây dựng theo **nguyên tắc Đa hình (Polymorphism)** và **Lập trình theo Hợp đồng (Design by Contract)**. Mục tiêu là để trừu tượng hóa hoàn toàn chi tiết kết nối của từng loại CSDL. Lớp Service ở trên chỉ cần biết "tôi muốn kết nối" và "tôi muốn lấy dữ liệu", mà không cần quan tâm làm thế nào để xây dựng chuỗi kết nối cho PostgreSQL hay làm thế nào để đọc một file Excel.

#### **3.1.1. `BaseConnector` & `BaseConnectionParams` (`base_connector.py`)**
Đây là nền tảng của toàn bộ hệ thống kết nối.

* **`BaseConnectionParams`:**
    * **Vai trò:** Là một Pydantic model, định nghĩa một cấu trúc dữ liệu **chuẩn hóa** cho mọi loại kết nối. Bất kể là kết nối tới MySQL hay một file, nó đều phải tuân theo cấu trúc cơ bản này (host, port, username, password...).
* **`BaseConnector`:**
    * **Vai trò:** Là một **Lớp cơ sở trừu tượng (Abstract Base Class - ABC)**, đóng vai trò như một "bản hợp đồng" mà tất cả các connector cụ thể khác đều phải tuân thủ.
    * **Phương thức Trừu tượng (Abstract Methods):**
        * `get_connection_url()`: Buộc các lớp con phải tự định nghĩa logic xây dựng chuỗi kết nối, vì mỗi CSDL có một định dạng riêng.
        * `get_table_size()`: Buộc các lớp con phải tự viết câu lệnh SQL để lấy dung lượng bảng, vì cú pháp này khác nhau giữa các hệ CSDL.
    * **Phương thức Chung (Shared Methods):**
        * Cung cấp sẵn các logic chung có thể tái sử dụng, chẳng hạn như `test_connection()` (bằng cách chạy `SELECT 1`), `list_tables()`, `get_columns()` (sử dụng SQLAlchemy Inspector), giúp tránh lặp code.

#### **3.1.2. Các Connector Cụ thể (`PostgresConnector`, `MySQLConnector`)**
Đây là các lớp triển khai "bản hợp đồng" `BaseConnector` cho từng hệ CSDL cụ thể.

* **Vai trò:** Cung cấp logic đặc thù cho PostgreSQL và MySQL.
* **Triển khai `get_connection_url()`:**
    * **PostgreSQL:** Xây dựng chuỗi kết nối theo định dạng `postgresql+psycopg2://...`, đồng thời hỗ trợ tham số `sslmode`.
    * **MySQL:** Xây dựng chuỗi kết nối theo định dạng `mysql+pymysql://...`, hỗ trợ tham số `ssl_mode`.
* **Triển khai `get_table_size()`:**
    * **PostgreSQL:** Thực thi hàm `pg_total_relation_size()` của PostgreSQL.
    * **MySQL:** Truy vấn bảng `information_schema.TABLES` của MySQL.
    * Đây là ví dụ điển hình cho thấy sự cần thiết của phương thức trừu tượng trong `BaseConnector`.

#### **3.1.3. Connector Chuyên biệt (`FileConnector`)**
Đây là một ví dụ xuất sắc về tính linh hoạt của kiến trúc Connector.

* **Vai trò:** "Giả lập" một nguồn dữ liệu từ file (CSV, Excel) để tuân thủ theo "hợp đồng" `BaseConnector`, giúp các service có thể làm việc với file một cách thống nhất như khi làm việc với CSDL.
* **Cơ chế hoạt động:**
    * **Không dùng SQLAlchemy:** Không tạo ra `Engine` của SQLAlchemy (`get_engine()` trả về `None`).
    * **Đọc và Giữ trong Bộ nhớ:** Ngay khi được khởi tạo, nó sẽ đọc toàn bộ nội dung file vào một **Polars DataFrame** và lưu trữ trong `self._dataframe`.
    * **"Giả lập" các phương thức:** Các phương thức như `list_tables()`, `get_columns()`, `get_table_size()` không thực hiện truy vấn mạng. Thay vào đó, chúng thực hiện các thao tác trên DataFrame hoặc hệ thống file đã có sẵn (ví dụ: `os.path.basename`, `df.schema`, `os.path.getsize`).

---

### **3.2. Lớp Lưu trữ Metadata (`src/database/`)**

**Triết lý thiết kế:**
Lớp này tuân thủ chặt chẽ **Mô hình Repository (Repository Pattern)**. Toàn bộ logic nghiệp vụ (trong các Services) hoàn toàn không biết và không quan tâm đến chi tiết cách dữ liệu được lưu trữ hay truy vấn. Chúng chỉ giao tiếp với thế giới bên ngoài thông qua một cổng duy nhất là `MetadataRepository`.

#### **3.2.1. Các Mô hình Dữ liệu (Data Models - `models.py`)**
Đây là "bản thiết kế" (schema) cho CSDL metadata của ứng dụng, được định nghĩa bằng SQLAlchemy ORM. Các bảng chính bao gồm:

* **`Connection`:** Lưu trữ thông tin kết nối đến các nguồn dữ liệu. Mật khẩu được lưu dưới dạng chuỗi, nhưng lớp Service sẽ đảm bảo nó được mã hóa trước khi lưu.
* **`ProfilingRun`:** Lưu trữ thông tin tổng quan của mỗi lần chạy profiling (tên bảng, connection id, tổng số dòng...). Nó có một mối quan hệ một-nhiều với `ColumnProfile`.
* **`ColumnProfile`:** Một bảng lớn lưu trữ chi tiết từng chỉ số thống kê cho mỗi cột trong một lần chạy. Việc tách ra bảng riêng giúp tối ưu hóa truy vấn và cấu trúc.
* **`DataQualityRule`:** Lưu định nghĩa của các quy tắc chất lượng dữ liệu do người dùng tạo ra (loại quy tắc, cột áp dụng, tham số JSON...).
* **`DataQualityResult`:** Lưu kết quả (pass/fail, số dòng lỗi) của một quy tắc cụ thể trong một lần chạy profiling. Nó liên kết đến cả `ProfilingRun` và `DataQualityRule`.
* **`DataRelationship`:** Lưu các mối quan hệ khóa ngoại được `RelationshipAnalyzer` phát hiện ra.

**Đặc điểm nổi bật:**
* **Quan hệ rõ ràng:** Sử dụng `relationship` và `back_populates` để định nghĩa các mối quan hệ hai chiều, giúp việc truy vấn lồng nhau (eager loading) trở nên hiệu quả.
* **Xóa theo tầng (Cascade Delete):** Cấu hình `cascade="all, delete-orphan"` đảm bảo rằng khi một bản ghi cha bị xóa (ví dụ: một `Connection`), tất cả các bản ghi con liên quan (profiling runs, DQ rules...) cũng sẽ tự động bị xóa, duy trì tính toàn vẹn của dữ liệu.

#### **3.2.2. `MetadataRepository` (`repository.py`)**
* **Vai trò:** Là lớp **cổng giao tiếp duy nhất (Gateway)** tới CSDL metadata. Nó đóng gói toàn bộ logic tương tác với SQLAlchemy Session và các hàm CRUD, cung cấp các phương thức nghiệp vụ rõ ràng cho lớp Service.
* **Quản lý Session:** Tự quản lý vòng đời của `Session` (lấy session từ `get_db_session`, thực hiện thao tác, `commit` hoặc `rollback`, và cuối cùng là `close`). Điều này đảm bảo tài nguyên CSDL được quản lý đúng cách và giúp các service không cần bận tâm về nó.
* **Giao tiếp qua CRUD:** Ủy quyền các thao tác CSDL cấp thấp cho một module `crud` (không được cung cấp nhưng có thể suy ra sự tồn tại). Ví dụ, `save_connection` trong repository sẽ gọi `crud.create_connection`.
* **Logic Tái cấu trúc Dữ liệu (`_convert_run_to_dict`):** Cung cấp một phương thức nội bộ để chuyển đổi đối tượng `ProfilingRun` phức tạp (bao gồm cả các `ColumnProfile` liên quan) thành một cấu trúc dictionary phẳng và chuẩn hóa, sẵn sàng để các service sử dụng hoặc lưu trữ dưới dạng JSON.
* **Logic đọc ưu tiên file (`get_run_by_id`):** Phương thức này thể hiện một logic nghiệp vụ quan trọng: khi được yêu cầu lấy một `run`, nó sẽ **ưu tiên tìm và đọc file JSON từ thư mục "golden_snapshots" trước**. Nếu không tìm thấy file, nó mới truy vấn vào CSDL. Điều này cho phép "Golden Snapshots" ghi đè lên các bản ghi lịch sử, đảm bảo tính nhất quán.

Chắc chắn rồi. Dựa trên mã nguồn UI bạn đã cung cấp, đây là tài liệu cho **Nhóm 4**, tập trung vào mục đích, cách sử dụng và các tính năng quan trọng của từng trang giao diện dưới góc nhìn của người dùng là một Nhà phân tích Dữ liệu (Data Analyst).

---
## **Phần 4: Giao diện Người dùng (UI Layer - `src/ui/`)**

**Triết lý thiết kế:**
Giao diện người dùng của DA-ODA được xây dựng hoàn toàn bằng **Streamlit**, hướng đến trải nghiệm tương tác, nhanh chóng và tập trung vào việc cung cấp các insight có giá trị cho người dùng cuối. UI đóng vai trò là "mặt tiền", giúp người dùng khai thác sức mạnh của các services và core engine ở backend một cách trực quan. Toàn bộ trạng thái của phiên làm việc (kết quả profiling, dữ liệu mẫu...) được quản lý thông qua `st.session_state` để đảm bảo tính nhất quán và liền mạch giữa các trang.

---

### **4.1. Trang Profiler & Kết quả Phân tích (`1_Profiler.py` & `profiler_results_view.py`)**

**Mục đích:** Đây là trang chính và là điểm khởi đầu của ứng dụng. Tại đây, người dùng kết nối đến nguồn dữ liệu, lựa chọn bảng, cấu hình việc lấy mẫu và kích hoạt quy trình phân tích dữ liệu toàn diện.

**Cách dùng (Luồng làm việc điển hình):**
1.  **Kết nối:** Sử dụng form kết nối để kết nối tới CSDL hoặc tải lên một file dữ liệu (CSV, Excel).
2.  **Lựa chọn:** Chọn Schema (nếu có) và bảng dữ liệu cần phân tích.
3.  **Cấu hình lấy mẫu:** Xem trước các thông tin của bảng (tổng số dòng, kích thước) và chọn một trong các chiến lược lấy mẫu:
    * **Full Load:** Tải toàn bộ bảng (nếu nhỏ hơn giới hạn) hoặc giới hạn số dòng đầu tiên.
    * **Adaptive:** Lấy mẫu ngẫu nhiên, hiệu năng cao.
    * **Time-based:** Lọc dữ liệu theo một khoảng thời gian.
4.  **Chạy Phân tích:** Bấm nút "Run Profiling" để bắt đầu.

**Khu vực Hiển thị Kết quả (`profiler_results_view.py`):**
Sau khi phân tích hoàn tất, một báo cáo tương tác và chi tiết sẽ được hiển thị, bao gồm các khu vực chính:

* **Bảng điểm Sức khỏe (Health Score Card):** Cung cấp cái nhìn tổng quan tức thì về chất lượng dữ liệu thông qua 4 chỉ số chính: **Điểm Sức khỏe**, **Số vấn đề cần xử lý**, **Số dòng trùng lặp**, và **Số cột có vấn đề**.
* **Kế hoạch Hành động (Action Plan):** Liệt kê các vấn đề được phát hiện theo mức độ ưu tiên (Nghiêm trọng, Cảnh báo), giải thích rõ vấn đề và đưa ra gợi ý khắc phục.
* **Sơ lược các Cột (Columns Overview):** Một bảng tóm tắt cực kỳ mạnh mẽ, cho phép người dùng lướt nhanh qua tất cả các cột. Mỗi dòng hiển thị tên cột, kiểu dữ liệu, tỷ lệ % NULL, vấn đề chính được phát hiện, và một **biểu đồ sparkline** trực quan hóa sự phân phối của dữ liệu.
* **Phân tích Chi tiết (Detailed Analysis):** Khu vực tương tác cho phép chọn một cột cụ thể để xem xét sâu hơn:
    * **Trực quan hóa (dành cho cột số):** Tự động hiển thị 2 biểu đồ histogram song song (một biểu đồ toàn cảnh và một biểu đồ đã zoom, loại bỏ các giá trị ngoại lai) và một biểu đồ hộp (box plot).
    * **Thống kê & Insights:** Hiển thị các bảng thống kê chi tiết, các cảnh báo và gợi ý (Insights) do hệ thống tự động phát hiện, và kết quả của các quy tắc DQ được áp dụng cho cột đó.

**Lưu ý:**
* Nút **"Tải về mẫu (CSV)"** cho phép người dùng tải về chính xác DataFrame đã được sử dụng cho quá trình phân tích, đảm bảo tính tái lập.
* Kết quả profiling thành công sẽ được lưu vào `st.session_state`, cho phép các trang khác như "Phân tích Tương quan" sử dụng lại mà không cần chạy lại.

---

### **4.2. Trang Phân tích Tương quan (`pages/1_Correlation_Analysis.py`)**

**Mục đích:** Khám phá và trực quan hóa mối quan hệ tương quan tuyến tính (Pearson) giữa các cột dữ liệu số của bảng đã được phân tích.

**Cách dùng:**
1.  Điều kiện tiên quyết là phải chạy profiling thành công trên trang "Profiler".
2.  Khi truy cập trang này, nó sẽ tự động tải dữ liệu từ `session_state`.
3.  Hệ thống sẽ **tự động gợi ý** các cột số phù hợp để phân tích, thông minh loại bỏ các cột ID, mã...
4.  Người dùng có thể tùy chỉnh lại danh sách cột và điều chỉnh **ngưỡng tương quan mạnh**.
5.  Bấm "Chạy Phân tích Tương quan" để xem kết quả.

**Kết quả:**
* Một **biểu đồ nhiệt (heatmap)** tương tác hiển thị ma trận tương quan.
* Một danh sách tóm tắt các **cặp tương quan mạnh** (cả tương quan thuận và nghịch) đã vượt ngưỡng do người dùng thiết lập.

---

### **4.3. Trung tâm Chất lượng Dữ liệu (`pages/3_Data_Quality_Center.py`)**

**Mục đích:** Là một trung tâm chỉ huy toàn diện cho mọi hoạt động liên quan đến chất lượng dữ liệu, từ việc định nghĩa quy tắc cho đến thực thi và làm sạch dữ liệu.

**Cách dùng (Giao diện 2 tab):**
* **Tab 1: Quản lý Quy tắc:**
    * **Xem và Xóa:** Người dùng chọn một bảng để xem danh sách các quy tắc hiện có, xem chi tiết tham số và xóa quy tắc (với cơ chế xác nhận 2 bước để tránh xóa nhầm).
    * **Thêm mới:** Một form động cho phép tạo quy tắc mới. Giao diện sẽ tự động thay đổi để hiển thị các trường nhập liệu phù hợp với từng loại quy tắc (ví dụ: hiển thị ô nhập liệu cho `cross_column_logic` hay các dropdown chọn bảng/cột cho `foreign_key`).
* **Tab 2: Chạy Kiểm tra & Xem Kết quả:**
    * **Thực thi:** Người dùng chọn bảng và bấm "Chạy tất cả kiểm tra DQ". Hệ thống sẽ chạy lại một quy trình profiling nhanh ở backend để lấy dữ liệu mới nhất và áp dụng tất cả các quy tắc.
    * **Xem kết quả:** Kết quả được hiển thị rõ ràng với trạng thái PASS (xanh) hoặc FAIL (đỏ). Với các quy tắc FAIL, người dùng có thể bấm **"Xem Chi tiết"** để mở một dialog (`st.dialog`) hiển thị các dòng dữ liệu đã vi phạm.
    * **Công cụ Làm sạch & Trích xuất:** Đây là một tính năng mạnh mẽ. Người dùng có thể chọn các quy tắc đã FAIL mà họ muốn thực thi. Sau khi bấm **"Tạo dữ liệu sạch"**, hệ thống sẽ loại bỏ các dòng vi phạm và cung cấp nút **"Tải về file CSV đã làm sạch"**.

---

### **4.4. Trang Khám phá Mối quan hệ (`pages/2_Relationship_Explorer.py`)**

**Mục đích:** Tự động phân tích schema của CSDL để phát hiện các mối quan hệ khóa ngoại (dù đã được định nghĩa hay chưa) và trực quan hóa chúng dưới dạng biểu đồ thực thể-quan hệ (ERD).

**Cách dùng:**
1.  Sau khi kết nối CSDL ở trang Profiler, người dùng truy cập trang này.
2.  Chọn Schema cần phân tích.
3.  Bấm nút **"Phát hiện & Cập nhật Mối quan hệ"**.

**Kết quả:**
* **Tab Bảng chi tiết:** Hiển thị danh sách các mối quan hệ tìm thấy dưới dạng bảng.
* **Tab Biểu đồ ERD:** Tự động vẽ một biểu đồ ERD tương tác bằng `graphviz`. Biểu đồ được thiết kế thông minh, chỉ hiển thị các cột có liên quan và làm nổi bật các cột khóa chính/khóa ngoại để dễ dàng theo dõi.

---

### **4.5. Trang Quản lý & So sánh Snapshot (`pages/4_Snapshot_Comparison.py` & `pages/5_Snapshot_Management.py`)**

**Mục đích:** Cung cấp các công cụ để quản lý và so sánh các "Golden Snapshots" - các phiên bản baseline quan trọng của dữ liệu đã được người dùng ghim lại.

* **Trang Quản lý (`5_Snapshot_Management.py`):**
    * **Cách dùng:** Hiển thị một bảng liệt kê tất cả các Golden Snapshots đã lưu. Người dùng có thể chọn một snapshot từ danh sách và thực hiện hành động **Xóa**.
    * **Lưu ý:** Giao diện tích hợp dialog xác nhận để đảm bảo việc xóa là có chủ đích.

* **Trang So sánh (`4_Snapshot_Comparison.py`):**
    * **Cách dùng:**
        1.  Người dùng chọn hai Baseline (A và B) từ hai dropdown.
        2.  Bấm nút "So sánh" để kích hoạt `ComparisonService` ở backend.
    * **Kết quả:**
        * **Thay đổi Cấu trúc:** Hiển thị rõ ràng số lượng và danh sách các **cột được thêm vào** và **cột bị xóa đi**.
        * **Thay đổi Toàn cục:** Sử dụng các bảng so sánh chi tiết để chỉ ra sự khác biệt trong "Thông tin Bảng", "Báo cáo Sức khỏe", và "Kết quả Quy tắc DQ". Giao diện có khả năng phân tích sâu vào các danh sách để làm nổi bật các mục đã được thêm, xóa, hoặc thay đổi nội dung.