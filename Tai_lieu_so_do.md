

## Cấu trúc Thư mục Dự án DA-ODA (Chi tiết)

```
.
├── alembic/                     # -> Quản lý các phiên bản migration cho CSDL metadata.
│   ├── env.py                   # -> File cấu hình môi trường Alembic, định nghĩa cách kết nối CSDL.
│   ├── README                   # -> Hướng dẫn sử dụng thư mục alembic.
│   ├── script.py.mako           # -> Template để tạo ra các file migration mới.
│   └── versions/                # -> Nơi chứa các file migration script theo từng phiên bản.
│       └── bd3e31052df8_initial_migration.py # -> Một file migration cụ thể.
├── alembic.ini                  # -> File cấu hình chính của Alembic.
├── config/                      # -> Quản lý toàn bộ cấu hình của ứng dụng.
│   ├── __init__.py
│   ├── config.py                # -> Quản lý chuỗi kết nối và các tham số cho CSDL NGUỒN.
│   ├── metrics_config.py        # -> Sổ đăng ký hành vi của các chỉ số (ví dụ: cao hơn là tốt hơn/tệ hơn).
│   └── profiling_settings.py    # -> Quản lý các tham số profiling (sample_size, ngưỡng...).
├── docker-compose.yml           # -> Định nghĩa các service (app, db) để chạy dự án với Docker.
├── Dockerfile                   # -> Chỉ dẫn để build image cho ứng dụng Python.
├── generate_key.py              # -> Script tạo khóa mã hóa cho việc lưu trữ connection credentials.
├── golden_snapshots/            # -> Nơi lưu trữ các file JSON của Baseline/Golden Snapshot.
│   ├── 39_golden_snapshot.json
│   └── 41_golden_snapshot.json
├── init-db/                     # -> Chứa các script SQL để khởi tạo CSDL lần đầu.
│   ├── init_mysql.sql
│   └── init.sql
├── logs/                        # -> Thư mục chứa file log của ứng dụng.
│   └── da_oda.log
├── metadata.db                  # -> Database SQLite cục bộ, lưu trữ metadata của ứng dụng.
├── poetry.lock                  # -> File khóa phiên bản chính xác của các thư viện.
├── pyproject.toml               # -> File chính của dự án Poetry, định nghĩa dependencies và thông tin dự án.
├── README.md                    # -> Hướng dẫn tổng quan về dự án.
├── scripts/                     # -> Chứa các script tiện ích cho việc phát triển và vận hành.
│   ├── init_db.py               # -> Script Python để khởi tạo hoặc làm mới CSDL metadata.
│   ├── run_golden_test.py
│   ├── run_integration_tests.sh # -> Script để chạy các bài kiểm thử tích hợp.
│   ├── setup_dev_db.sh          # -> Script thiết lập database cho môi trường dev.
│   ├── test_connection.py
│   └── verify_connectors.py
├── sodo.md                      # -> File sơ đồ, mô tả kiến trúc dự án (chính là file này).
├── src/                         # -> Thư mục chứa toàn bộ mã nguồn chính của ứng dụng.
│   ├── __init__.py
│   ├── core/                    # 🎯 LÕI: Vùng hiệu năng cao, chỉ làm việc với Polars, không phụ thuộc UI.
│   │   ├── __init__.py
│   │   ├── data_quality_engine.py   # -> "Động cơ" thực thi các quy tắc Chất lượng Dữ liệu (DQ).
│   │   ├── engine/              # -> Các thành phần cốt lõi của việc phân tích.
│   │   │   ├── analyzer.py      # -> Logic phân tích tương quan trên Polars DataFrame.
│   │   │   ├── profiler.py      # -> Logic tính toán các chỉ số profiling bằng Polars.
│   │   │   └── sampler.py       # -> Logic lấy mẫu dữ liệu tốc độ cao, hỗ trợ đa CSDL.
│   │   └── relationship_analyzer.py # -> "Động cơ" phân tích, khám phá mối quan hệ giữa các bảng.
│   ├── database/                # 📦 Vùng quản lý CSDL METADATA của chính ứng dụng.
│   │   ├── __init__.py
│   │   ├── crud.py              # -> Các hàm thao tác CSDL cấp thấp (Create, Read, Update, Delete).
│   │   ├── models.py            # -> "Bản thiết kế" (schema) các bảng trong CSDL metadata bằng SQLAlchemy.
│   │   ├── repository.py        # -> Lớp "Repository" - Cổng giao tiếp duy nhất giữa Service và CSDL.
│   │   ├── session.py           # -> Quản lý session làm việc với CSDL metadata.
│   │   └── utils.py             # -> Các tiện ích riêng của module database.
│   ├── periphery/               # 🔌 NGOẠI VI: Vùng giao tiếp với các hệ thống bên ngoài.
│   │   ├── adapters/            # -> Các "bộ chuyển đổi" dữ liệu giữa các lớp.
│   │   │   ├── profiling_adapter.py # -> Làm giàu và định dạng kết quả thô từ Lõi thành JSON cho UI.
│   │   │   └── viz_adapter.py       # -> Chuyển đổi dữ liệu để phù hợp với các thư viện vẽ biểu đồ.
│   │   └── connectors/          # -> Logic kết nối đến các CSDL nguồn (Postgres, MySQL, Files...).
│   │       ├── __init__.py
│   │       ├── base_connector.py    # -> Lớp cơ sở trừu tượng cho tất cả các connector.
│   │       ├── file_connector.py
│   │       ├── mysql_connector.py
│   │       └── postgres_connector.py
│   ├── services/                # 🌉 CẦU NỐI: Lớp điều phối nghiệp vụ, kết nối UI và Lõi/Database.
│   │   ├── __init__.py
│   │   ├── comparison_service.py    # -> Service xử lý logic so sánh giữa các snapshot.
│   │   ├── connection_service.py    # -> Quản lý việc tạo, lưu và hủy kết nối đến CSDL nguồn.
│   │   ├── crypto_service.py        # -> Service mã hóa và giải mã thông tin nhạy cảm (mật khẩu).
│   │   ├── file_profiling_service.py # -> Service chuyên biệt cho việc phân tích file.
│   │   ├── governance_service.py    # -> Điều phối các tác vụ Quản trị Dữ liệu (DQ, Mối quan hệ).
│   │   └── profiling_service.py     # -> "Nhạc trưởng" điều phối toàn bộ quy trình profiling cho CSDL.
│   ├── ui/                      # 🖥️ GIAO DIỆN NGƯỜI DÙNG: Toàn bộ code liên quan đến Streamlit.
│   │   ├── __init__.py
│   │   ├── 1_Profiler.py        # -> Trang chính của ứng dụng.
│   │   ├── app.py               # -> Chứa các hàm khởi tạo và cache service provider dùng chung toàn ứng dụng.
│   │   ├── components/          # -> Các thành phần UI nhỏ, có thể tái sử dụng.
│   │   │   ├── connection_form.py
│   │   │   └── file_uploader_form.py
│   │   ├── pages/               # -> Các trang phụ của ứng dụng Streamlit.
│   │   │   ├── __init__.py
│   │   │   ├── 1_Correlation_Analysis.py
│   │   │   ├── 2_Relationship_Explorer.py
│   │   │   ├── 3_Data_Quality_Center.py
│   │   │   ├── 4_Snapshot_Comparison.py
│   │   │   └── 5_Snapshot_Management.py
│   │   ├── state/               # -> Thư mục quản lý trạng thái giao diện người dùng.
│   │   │   └── profiler_state.py    # -> Quản lý session state tập trung cho trang Profiler.
│   │   └── views/               # -> Các phần hiển thị lớn, phức tạp của một trang.
│   │       ├── correlation_view.py
│   │       ├── profiler_controls.py   # -> Giao diện khu vực điều khiển (chọn bảng, nút bấm...).
│   │       └── profiler_results_view.py    # -> Giao diện khu vực hiển thị kết quả profiling.
│   └── utils/                   # -> Các tiện ích dùng chung không thuộc nghiệp vụ cụ thể.
│       ├── __init__.py
│       ├── logger.py            # -> Cấu hình hệ thống logging.
│       ├── sparkline_generator.py
│       └── viz_helpers.py
└── tests/                       # -> Thư mục chứa các bài kiểm thử (tests).
    ├── __init__.py
    ├── core/
    │   ├── __init__.py
    │   └── engine/
    │       ├── __init__.py
    │       └── test_profiler.py
    ├── periphery/
    │   ├── __init__.py
    │   └── adapters/
    │       ├── __init__.py
    │       └── test_profiling_adapter.py
    └── test_end_to_end.py
```