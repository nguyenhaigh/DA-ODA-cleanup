
<p align="center">
  <h1 align="center">DA-ODA (Data Analytics - On-Demand Analysis)</h1>
</p>

<p align="center">
    <img src="https://img.shields.io/badge/python-3.11+-blue.svg" alt="Python Version">
    <img src="https://img.shields.io/badge/framework-Streamlit-red.svg" alt="Framework">
    <img src="https://img.shields.io/badge/engine-Polars-purple.svg" alt="Engine">
    <img src="https://img.shields.io/badge/license-MIT-green.svg" alt="License">
    </p>

## 🎯 DA-ODA: Nền tảng Phân tích Dữ liệu Hiệu suất cao

**DA-ODA** được thiết kế để giải quyết "nỗi đau" lớn nhất của Nhà phân tích Dữ liệu trong kỷ nguyên dữ liệu lớn: **Tốc độ** và **Tính Toàn vẹn**. Nó biến quy trình phân tích thủ công, chậm chạp và dễ sai sót thành một trải nghiệm **tốc độ, tự động và đáng tin cậy**.

Nền tảng này kết hợp sức mạnh xử lý song song của **Polars**, một hệ thống **Baseline/Snapshot** mạnh mẽ để theo dõi sự biến động dữ liệu, và một **Trung tâm Chất lượng Dữ liệu** trực quan.

---

## ✨ Tính năng Cốt lõi: Hiệu suất & Độ Tin cậy

| Tính năng | Giải quyết Vấn đề Nghiệp vụ (Pain Point Killer) | Mã nguồn Củng cố |
| :--- | :--- | :--- |
| ⚡ **Phân tích Hiệu suất Cao** | Loại bỏ tình trạng tắc nghẽn I/O & CPU khi làm việc với các mẫu dữ liệu lớn. Cho phép profiling **toàn diện** thay vì chỉ dựa trên mẫu nhỏ. | `src/core/engine/profiler.py` |
| ⚖️ **Hệ thống Baseline & So sánh Sâu** | **Điểm mạnh độc nhất:** So sánh chi tiết hai **Golden Snapshots** (Baseline A vs. B) để xác định sự khác biệt về **cấu trúc**, **chỉ số toàn cục** và **kết quả DQ**. | `src/services/comparison_service.py` |
| ✅ **Trung tâm Chất lượng Dữ liệu** | Cung cấp khả năng **phát hiện, làm sạch, và trích xuất** các dòng lỗi theo quy tắc (ví dụ: `cross_column_logic`). | `src/core/data_quality_engine.py` |
| 🗺️ **Khám phá Mối quan hệ Tự động** | Tự động xác thực và trực quan hóa mô hình dữ liệu (ERD), giảm sự phụ thuộc vào tài liệu CSDL lỗi thời. | `src/ui/pages/2_Relationship_Explorer.py` |
| 🔄 **Quản lý Vòng đời Dữ liệu Mẫu** | Dễ dàng tải về chính xác **DataFrame mẫu** đã được sử dụng cho phân tích, đảm bảo tính **tái lập (Reproducibility)** của insight. | `src/core/engine/sampler.py` |

---

## 🛠️ Ngăn xếp Công nghệ (Technology Stack)

| Lĩnh vực | Công nghệ |
| :--- | :--- |
| **Backend & Core Engine** | `Polars`, `SQLAlchemy`, `Pydantic`, `ConnectorX` |
| **Giao diện người dùng** | `Streamlit` |
| **Nguồn dữ liệu được hỗ trợ** | **PostgreSQL**, **MySQL**, **Files** (CSV, Excel) |
| **Cơ sở dữ liệu (Metadata)** | `SQLite` (Development), `PostgreSQL` (Production-ready) |
| **Quản lý Dependencies & Môi trường** | `Poetry`, `Docker`, `Docker Compose` |
| **Quản lý Schema CSDL** | `Alembic` |
| **Kiểm thử** | `pytest` |

---

## 🏛️ Kiến trúc Hệ thống (Architecture)

DA-ODA được xây dựng dựa trên các nguyên tắc **Kiến trúc Sạch (Clean Architecture)** để tách biệt rõ ràng giữa Business Logic (Core) và các thành phần ngoại vi (UI, Database, Connectors).

* **Tách biệt Vấn đề (Separation of Concerns):** Logic phân tích (`src/core`) hoàn toàn độc lập với UI (`src/ui`) và CSDL Metadata (`src/database`).
* **Dependency Injection (DI):** Tất cả các service đều được "tiêm" các dependency cần thiết, giúp dễ dàng kiểm tra và thay thế.

| Thư mục | Mô tả Chức năng |
| :--- | :--- |
| `src/core/` | **🎯 LÕI:** Vùng Hiệu suất cao, chứa logic Polars cho Profiling, Sampling, Analysis. |
| `src/database/` | **📦 LƯU TRỮ:** Quản lý CSDL Metadata bằng SQLAlchemy ORM, Repository. |
| `src/periphery/` | **🔌 NGOẠI VI:** Giao tiếp với thế giới bên ngoài. Chứa các **Connectors** (`Postgres`, `MySQL`, `File`) và **Adapters**. |
| `src/services/` | **🌉 CẦU NỐI:** Lớp điều phối logic nghiệp vụ giữa Core và UI. |
| `src/ui/` | **🖥️ GIAO DIỆN:** Giao diện người dùng Streamlit và quản lý trạng thái. |
| `tests/` | **🧪 KIỂM THỬ:** Đảm bảo tính chính xác của các phép tính và luồng nghiệp vụ. |

---
## 📂 Cấu trúc Thư mục

```
.
├── alembic/                     # Quản lý migration CSDL metadata
├── config/                      # Cấu hình runtime (DB, metrics, profiling params)
├── golden_snapshots/            # Lưu trữ các file JSON của Baseline/Golden Snapshot
├── metadata.db                  # Database SQLite cục bộ (Metadata)
├── pyproject.toml               # Định nghĩa Dependencies (Poetry)
└── src/                         # MÃ NGUỒN CHÍNH
    ├── core/                    # 🎯 LÕI: Vùng Hiệu suất cao (Polars)
    │   ├── data_quality_engine.py   # -> "Động cơ" thực thi các quy tắc Chất lượng Dữ liệu (DQ).
    │   ├── engine/              # -> Các thành phần cốt lõi của việc phân tích.
    │   │   ├── analyzer.py      # -> Logic phân tích tương quan trên Polars DataFrame.
    │   │   ├── profiler.py      # -> Logic tính toán các chỉ số profiling bằng Polars.
    │   │   └── sampler.py       # -> Logic lấy mẫu dữ liệu tốc độ cao, hỗ trợ đa CSDL.
    │   └── relationship_analyzer.py # -> "Động cơ" phân tích, khám phá mối quan hệ giữa các bảng.
    ├── database/                # 📦 Quản lý CSDL METADATA
    ├── periphery/               # 🔌 NGOẠI VI: Connectors & Adapters
    ├── services/                # 🌉 CẦU NỐI: Logic Nghiệp vụ (Profiling, DQ, Comparison)
    └── ui/                      # 🖥️ GIAO DIỆN NGƯỜI DÙNG (Streamlit Pages)
```
## 🚀 Hướng dẫn Bắt đầu Nhanh

### 1. Triển khai (Production/Demo với Docker)

Sử dụng Docker Compose để triển khai toàn bộ ứng dụng và các dependency cần thiết trong vài phút.

1.  **Thiết lập Khóa Bảo mật** (Bắt buộc để mã hóa mật khẩu kết nối):
    ```bash
    # Lệnh này sẽ tạo ra một khóa mã hóa, sao chép nó.
    python generate_key.py
    # Sau đó, tạo file .env và thêm khóa vào: ENCRYPTION_KEY=your_generated_key
    ```
   
2.  **Xây dựng và Khởi động:**
    ```bash
    docker-compose up -d --build
    ```
   
3.  **Truy cập:** Mở trình duyệt và truy cập `http://localhost:8501`.

### 2. Môi trường Phát triển (Local Development)

Sử dụng Poetry để quản lý dependencies.

1.  **Cài đặt Dependencies:**
    ```bash
    poetry install
    ```
   
2.  **Khởi tạo CSDL Metadata:**
    ```bash
    poetry run python scripts/init_db.py
    ```
   
3.  **Khởi chạy Giao diện:**
    ```bash
    poetry run streamlit run src/ui/1_Profiler.py
    ```
   

---

## 🤝 Đóng góp (Contributing)

Chúng tôi hoan nghênh các đóng góp! Vui lòng tạo một "Issue" để thảo luận về các thay đổi lớn hoặc tạo một "Pull Request" cho các bản sửa lỗi và cải tiến nhỏ.

## 📄 Giấy phép (License)

Dự án này được cấp phép theo **Giấy phép MIT**. Xem file `LICENSE` để biết thêm chi tiết.

## 📧 Liên hệ

Được tạo bởi **[Tên của bạn]** - [Link tới LinkedIn hoặc GitHub của bạn]
