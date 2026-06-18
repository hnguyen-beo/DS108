# DS108 — Thu thập & Tiền xử lý Dữ liệu Tài chính Doanh nghiệp cho Dự đoán Lợi nhuận

> Đồ án môn **DS108 - Tiền xử lý và Xây dựng Bộ dữ liệu**
> 
> **GVHD:** TS. Nguyễn Gia Tuấn Anh — CN. Trần Quốc Khánh
>
> | Họ và tên | MSSV |
> |---|---|
> | Võ Hạnh Nguyên | 24521218 |
> | Phan Thị Ánh Nguyệt | 24521221 |

---

## 1. Giới thiệu

Đồ án xây dựng một **pipeline hoàn chỉnh** từ thu thập đến tiền xử lý dữ liệu **Báo cáo Kết quả Kinh doanh** (Income Statement) theo quý của **23 doanh nghiệp** niêm yết trên NYSE/NASDAQ thông qua **Alpha Vantage API**, nhằm tạo ra bộ dữ liệu sẵn sàng cho bài toán **dự đoán lợi nhuận quý tiếp theo**.

**Nguồn dữ liệu:** 
- Corporate Financials Statements (AlphaVantage) (https://www.kaggle.com/datasets/emranalbiek/companies-financial-income-statements/data)
- Alpha Vantage Financial API (Income Statement, 23 doanh nghiệp NYSE/NASDAQ)

**Biến mục tiêu:**
- `next_quarter_net_income` — lợi nhuận ròng quý tới (bài toán **hồi quy**)
- `profit_increase_next` — quý tới lợi nhuận tăng hay không (bài toán **phân loại**)

Toàn bộ pipeline được thiết kế với nguyên tắc **chống Data Leakage** nghiêm ngặt: chia dữ liệu theo thời gian, mọi tham số xử lý (imputation, scaling, winsorization, feature selection...) chỉ học từ tập train rồi áp dụng cho tập test.

Project gồm 3 notebook chạy theo thứ tự:

| Thứ tự | Notebook | Mục đích |
|---|---|---|
| 1 | `notebooks/01_data_collection.ipynb` | Gọi Alpha Vantage API, thu thập Income Statement của 23 mã cổ phiếu → lưu `data/raw/Data_DS108_raw.csv` |
| 2 | `notebooks/02_data_cleaning_and_imputation.ipynb` | Làm sạch, feature engineering, tạo biến mục tiêu, xử lý outlier, split train/test, imputation, scaling, feature selection → lưu `data/processed/Data_DS108_processed.csv` và `data/processed/train_after_feature_selection.csv` |
| 3 | `notebooks/03_exploratory_data_analysis.ipynb` | Phân tích khám phá dữ liệu (EDA) trước và sau xử lý, dùng cả file raw và file processed ở trên |

> ⚠️ Phải chạy đúng thứ tự **01 → 02 → 03** vì mỗi notebook sau phụ thuộc vào file đầu ra của notebook trước.

---

## 2. Cấu trúc thư mục

```
project-root/
├── notebooks/
│   ├── 01_data_collection.ipynb
│   ├── 02_data_cleaning_and_imputation.ipynb
│   └── 03_exploratory_data_analysis.ipynb
├── requirements.txt
├── README.md                            # File hướng dẫn này
│
├── data/                                # (Tự sinh khi chạy code)
│   ├── raw/
│   │   ├── Data_DS108_raw.csv           # Output của notebook 01 (đầu vào của 02 & 03)
│   │   └── {SYMBOL}_income_quarterly.csv   # File raw từng công ty (23 file, backup)
│   └── processed/
│       ├── Data_DS108_processed.csv     # Output của notebook 02 (đầu vào của 03)
│       └── train_after_feature_selection.csv
│
└── logs/                                # (Tự sinh) File log quá trình crawl
    └── crawl.log
```

> Các thư mục `data/` và `logs/` **không cần tạo tay** — code sẽ tự tạo bằng `Path(...).mkdir(parents=True, exist_ok=True)`.

---

## 3. Yêu cầu môi trường

- **Python**: 3.10 trở lên (khuyến nghị 3.11; code dùng cú pháp type hint mới như `pd.DataFrame | None`)
- **pip** (trình quản lý gói của Python)
- **Jupyter Notebook / JupyterLab** hoặc VS Code (extension Jupyter)
- Kết nối Internet (để notebook 01 gọi Alpha Vantage API)
- Một **API Key miễn phí** từ Alpha Vantage (xem hướng dẫn ở Mục 4)

---

## 4. Lấy API Key của Alpha Vantage (chỉ cần cho notebook 01)

1. Truy cập: https://www.alphavantage.co/support/#api-key
2. Nhập email và bấm **GET FREE API KEY**.
3. Hệ thống sẽ cấp ngay một chuỗi API Key (ví dụ: `ABCD1234XYZ`).
4. **Lưu lại key này** — sẽ cần nhập khi chạy notebook 01.

> ⚠️ **Lưu ý về giới hạn (Free tier):** Alpha Vantage free tier giới hạn **5 request/phút** và **~25 request/ngày**. Với 23 mã cổ phiếu, notebook 01 đã cấu hình `REQUEST_DELAY_SECONDS = 13` (delay 13 giây/request) để tránh vượt giới hạn — quá trình thu thập toàn bộ sẽ mất khoảng **5 phút**.
>
> Nếu log hiện `"Note"` (chạm rate limit), hãy chờ vài phút rồi chạy lại, hoặc đợi sang ngày sau, hoặc dùng API key Premium. Nếu log hiện `"Information"` (yêu cầu Premium) ở một số mã/endpoint, code sẽ tự ghi mã đó vào danh sách `failed` — có thể bỏ qua hoặc thử với key khác.

---

## 5. Cài đặt môi trường (từ A đến Z)

### Bước 5.1 — Lấy mã nguồn

Giải nén / clone project về máy, đặt 3 file `.ipynb`, `requirements.txt` và `README.md` đúng vào cấu trúc thư mục như Mục 2.

### Bước 5.2 — Tạo môi trường ảo (virtual environment)

Mở terminal/Command Prompt tại thư mục gốc của project (`project-root/`) và chạy:

**Windows (PowerShell / Command Prompt):**
```bash
python -m venv venv
venv\Scripts\activate
```

**macOS / Linux:**
```bash
python3 -m venv venv
source venv/bin/activate
```

Sau khi activate, dòng lệnh sẽ hiện tiền tố `(venv)`.

### Bước 5.3 — Cài đặt các thư viện cần thiết

Với `(venv)` đã được activate, chạy:

```bash
pip install --upgrade pip
pip install -r requirements.txt
```

Lệnh này sẽ cài đặt toàn bộ thư viện cần thiết: `pandas`, `numpy`, `scikit-learn`, `requests`, `matplotlib`, `seaborn`, `scipy`, `imbalanced-learn`, `jupyter`, v.v.

> 💡 Nếu chỉ muốn cài tối thiểu để chạy được 3 notebook, có thể cài nhanh nhóm thư viện chính:
> ```bash
> pip install pandas numpy scikit-learn requests matplotlib seaborn scipy imbalanced-learn jupyter
> ```

### Bước 5.4 — Cài Jupyter kernel cho venv (nếu chưa có)

```bash
pip install ipykernel
python -m ipykernel install --user --name=ds108-env
```

### Bước 5.5 — Mở Jupyter

```bash
jupyter lab
```
hoặc
```bash
jupyter notebook
```

Trình duyệt sẽ tự mở. Bạn cũng có thể mở thư mục project bằng **VS Code** (cài extension Jupyter) hoặc **Google Colab**.

> Nếu dùng Jupyter, nhớ chọn đúng **kernel** vừa tạo ở Bước 5.4 (`ds108-env`) trong menu Kernel → Change Kernel.

---

## 6. Cách chạy — CHẠY ĐÚNG THỨ TỰ 01 → 02 → 03

Ba notebook nối với nhau qua các file CSV, nên **bắt buộc chạy tuần tự**.

```
┌──────────────┐   sinh   ┌──────────────────────────┐   đọc   ┌──────────────┐
│ notebook 01  │ ───────► │  data/raw/...raw.csv      │ ──────► │ notebook 02  │
└──────────────┘          └──────────────────────────┘         └──────────────┘
                                                                        │ sinh
                                                                        ▼
                          ┌───────────────────────────────┐
                          │ data/processed/...processed.csv │
                          └───────────────────────────────┘
                                          │ đọc cả raw + processed
                                          ▼
                                  ┌──────────────┐
                                  │ notebook 03  │  (vẽ biểu đồ EDA)
                                  └──────────────┘
```

### ▶ Bước 1 — `01_data_collection.ipynb` (Thu thập)

1. Mở notebook, chạy các cell từ trên xuống (`Run > Run All Cells` hoặc `Shift+Enter` từng cell):
   - **Cài đặt thư viện** — import `requests`, `pandas`, `os`, `time`, `logging`, ...
   - **Cấu hình toàn cục** — khi chạy đến cell này, một ô nhập (`getpass`) sẽ hiện ra:
     ```
     Nhập API key của bạn:
     ```
     → **Dán API key đã lấy ở Mục 4 vào đây và nhấn Enter** (ký tự sẽ được ẩn, không hiển thị).
   - **Cấu hình logging** — tạo file log tại `logs/crawl.log`.
   - **Định nghĩa các cột thu thập** — ánh xạ 26 chỉ tiêu tài chính từ Income Statement.
   - **Hàm thu thập dữ liệu từ API** — định nghĩa `fetch_income_statement()`.
   - **Hàm crawl toàn bộ danh sách** — định nghĩa `crawl_all()`.
   - **Chạy pipeline thu thập dữ liệu** — cell cuối thực thi `crawl_all(SYMBOLS, period="quarterly")`, gọi API cho 23 mã, mỗi request cách nhau 13 giây (~5 phút tổng). Theo dõi tiến độ trên output cell hoặc trong `logs/crawl.log`.

**Kết quả đầu ra:**
- `data/raw/{SYMBOL}_income_quarterly.csv` — dữ liệu thô của từng công ty (23 file, backup)
- `data/raw/Data_DS108_raw.csv` — file tổng hợp toàn bộ 23 công ty (**input cho notebook 02 và 03**)
- `logs/crawl.log` — log quá trình crawl

> ✅ Kiểm tra thành công: log hiện `✅ HOÀN TẤT! Tổng N bản ghi từ 23 công ty.` và file `data/raw/Data_DS108_raw.csv` đã được tạo.

### ▶ Bước 2 — `02_data_cleaning_and_imputation.ipynb` (Tiền xử lý)

Chạy `Run All Cells`. Không cần nhập API key ở notebook này. Notebook tự đọc `data/raw/Data_DS108_raw.csv` và thực hiện pipeline 10 bước:

1. Đọc dữ liệu thô từ `data/raw/Data_DS108_raw.csv`
2. Làm sạch dữ liệu (loại bỏ dòng rác, chuẩn hoá kiểu dữ liệu, trùng lặp, quy đổi đơn vị USD...)
3. Feature Engineering (tính margin, tăng trưởng QoQ/YoY, rolling, lag...)
4. Tạo biến mục tiêu (`next_quarter_net_income`, `profit_increase_next` — dùng kỹ thuật shift -1)
5. Xử lý outlier (Sign-Log Transform)
6. Chia dữ liệu theo thời gian (Time-Series Split: train ≤ 2021, test > 2021)
7. Imputation (điền giá trị thiếu theo symbol + median ngành — fit trên train, áp dụng cho test)
8. Pipeline tiền xử lý sklearn: Sector Winsorization + StandardScaler (fit trên train)
9. Feature Selection (loại bỏ feature tương quan cao > 0.95 / phương sai thấp)
10. Tổng kết pipeline — in shape dữ liệu qua từng bước, checklist chống Data Leakage, và xác nhận file đã lưu

**Kết quả đầu ra:**
- `data/processed/Data_DS108_processed.csv` — toàn bộ dữ liệu sau xử lý
- `data/processed/train_after_feature_selection.csv` — dữ liệu train sau feature selection, kèm các biến `X_train_sel`, `X_test_sel`, `y_train`, `y_test` sẵn sàng cho model

> ⚠️ Notebook này dùng `PROJECT_ROOT = Path.cwd().parent`, nghĩa là nó giả định bạn đang chạy notebook **từ thư mục `notebooks/`** (đường dẫn tương đối tới `data/` được tính từ thư mục gốc project, là thư mục cha của `notebooks/`). Hãy mở/chạy file trực tiếp từ vị trí gốc trong cây thư mục như mô tả ở Mục 2, không di chuyển file `.ipynb` sang nơi khác.
>
> ✅ Kiểm tra thành công: cell cuối in `✅ PIPELINE SUMMARY` cùng bảng shape qua từng bước và checklist chống Data Leakage.

### ▶ Bước 3 — `03_exploratory_data_analysis.ipynb` (EDA)

Chạy `Run All Cells`. Notebook đọc **cả hai** file:
- `data/raw/Data_DS108_raw.csv` (dữ liệu **trước** xử lý — output của notebook 01)
- `data/processed/Data_DS108_processed.csv` (dữ liệu **sau** xử lý — output của notebook 02)

Pipeline thực hiện:

1. Import thư viện
2. Sector Mapping (gán 23 mã cổ phiếu vào 7 nhóm ngành)
3. Đọc dữ liệu (raw + processed)
4. EDA trước xử lý: thống kê mô tả, missing value, outlier (IQR, boxplot), phân phối & Q-Q plot, tương quan Pearson, panel data theo công ty, kiểm tra liên tục chuỗi thời gian, ranking doanh thu
5. EDA sau xử lý: kiểm tra nhanh, phân phối target, hiệu quả pipeline (before/after transform, correlation heatmap), phân phối feature theo công ty, time-series revenue & YoY growth, phân phối theo ngành
6. Tổng kết, so sánh trước/sau xử lý nhằm biện minh các quyết định ở notebook 02

**Kết quả:** các bảng thống kê và biểu đồ hiển thị trực tiếp trong notebook, đồng thời lưu thành các file `.png` (revenue ranking, outlier before/after, correlation heatmap, time-series, sector distribution, target distribution).

---

## 7. Tóm tắt thứ tự chạy nhanh

```bash
# 1. Tạo & kích hoạt môi trường ảo
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

# 2. Cài thư viện
pip install --upgrade pip
pip install -r requirements.txt

# 3. (Tuỳ chọn) Cài kernel riêng cho venv
pip install ipykernel
python -m ipykernel install --user --name=ds108-env

# 4. Mở Jupyter
jupyter notebook   # hoặc: jupyter lab

# 5. Chạy lần lượt (Run All), theo đúng thứ tự:
#    notebooks/01_data_collection.ipynb        → nhập API key khi được hỏi
#    notebooks/02_data_cleaning_and_imputation.ipynb
#    notebooks/03_exploratory_data_analysis.ipynb
```

---

## 8. Pipeline tổng quan

```
[01] Thu thập:   API → 23 công ty × báo cáo quý → raw CSV
        │
[02] Tiền xử lý: Đọc → Làm sạch (chuẩn hóa cột, duplicate, quy đổi USD)
        │        → Feature Engineering (margin, QoQ/YoY, rolling, lag)
        │        → Tạo target (shift -1) → Outlier (Sign-Log)
        │        → Split theo thời gian (≤2021 train / >2021 test)
        │        → Impute (theo symbol + median ngành) → Winsorize + Scale
        │        → Feature Selection (corr > 0.95, low variance) → processed CSV
        │
[03] EDA:        So sánh trước/sau xử lý → biện minh mọi quyết định ở bước 02
```

---

## 9. Đầu ra của dự án

| File | Sinh bởi | Mô tả |
|---|---|---|
| `data/raw/Data_DS108_raw.csv` | NB 01 | Dữ liệu thô gộp 23 công ty (input NB 02 & 03) |
| `data/raw/{SYMBOL}_income_quarterly.csv` | NB 01 | Backup từng công ty (23 file) |
| `data/processed/Data_DS108_processed.csv` | NB 02 | Dữ liệu đã qua toàn bộ pipeline (input NB 03) |
| `data/processed/train_after_feature_selection.csv` | NB 02 | Dữ liệu train sau feature selection |
| `logs/crawl.log` | NB 01 | Nhật ký quá trình thu thập |
| `*.png` | NB 03 | Các biểu đồ EDA |

---

## 10. Xử lý sự cố thường gặp

| Triệu chứng | Nguyên nhân & cách khắc phục |
|---|---|
| `FileNotFoundError: data/raw/Data_DS108_raw.csv` | Chưa chạy notebook 01, hoặc chạy 02 trước 01 → Chạy lại đúng thứ tự, kiểm tra file tồn tại trong `data/raw/`. |
| Log hiện `"Note"` / chạm rate limit | Gọi API quá nhanh hoặc hết quota ngày (vượt 5 req/phút hoặc ~25 req/ngày) → Chờ vài phút rồi chạy lại, hoặc đợi sang ngày khác, hoặc dùng API key Premium/khác. |
| Log hiện `"Information"` (yêu cầu Premium) | Một số mã/endpoint yêu cầu gói trả phí → Bỏ qua mã đó (code tự ghi vào `failed`) hoặc dùng key khác. |
| `ModuleNotFoundError` (ví dụ `No module named 'sklearn'`) | Chưa cài đúng thư viện hoặc chưa activate venv → Kiểm tra đã `activate` venv và đã chạy `pip install -r requirements.txt`. |
| Notebook chạy nhưng không thấy file `data/processed/...` | Đang chạy notebook từ thư mục khác `notebooks/`, khiến `Path.cwd().parent` trỏ sai thư mục gốc → Đảm bảo mở/chạy `.ipynb` đúng vị trí trong cấu trúc thư mục gốc ở Mục 2. |
| Biểu đồ không hiện ở notebook 03 | Chưa chạy notebook 02 (thiếu file processed) → Chạy 02 trước. |
| Cảnh báo `Time-series gaps` ở notebook 02 | Có công ty bị thiếu quý → YoY/rolling có thể lệch. Đây là **cảnh báo**, không phải lỗi (xem EDA Mục 6.5). |
| Lỗi version pandas/numpy | Dùng đúng version trong `requirements.txt`, hoặc tạo lại môi trường ảo sạch (xoá `venv/` và làm lại Bước 5.2–5.3). |

---

## 11. Ghi chú

- **Không commit API key** lên Git/GitHub. Code dùng `getpass` để nhập key lúc chạy, không lưu vào file.
- Thư mục `data/` và `logs/` nên được thêm vào `.gitignore` nếu dùng Git.
- Để dùng dữ liệu đầu ra cho mô hình ML, sau notebook 02 có thể nạp trực tiếp các biến đã sẵn:
  ```python
  from sklearn.ensemble import RandomForestClassifier
  clf = RandomForestClassifier(random_state=42, class_weight="balanced")
  clf.fit(X_train_sel, y_train)
  clf.score(X_test_sel, y_test)
  ```

---

## 12. Thông tin đồ án

- **Môn học:** DS108 - Tiền xử lý và Xây dựng bộ dữ liệu
- **GVHD:** TS. Nguyễn Gia Tuấn Anh - CN. Trần Quốc Khánh
- **Sinh viên thực hiện:**

| Họ và tên | MSSV |
|---|---|
| Võ Hạnh Nguyên | 24521218 |
| Phan Thị Ánh Nguyệt | 24521221 |

- **Nguồn dữ liệu:** Alpha Vantage Financial API (Income Statement, 23 doanh nghiệp NYSE/NASDAQ)
- **Biến mục tiêu:** `next_quarter_net_income` (hồi quy) và `profit_increase_next` (phân loại)
