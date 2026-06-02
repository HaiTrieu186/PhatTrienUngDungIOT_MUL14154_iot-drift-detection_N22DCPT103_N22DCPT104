# 🔍 IoT Concept Drift Detection
> Phát hiện drift trong hệ thống IoT bằng Online Learning — LSTM + ADWIN + Azure Cloud

**Môn học:** Phát triển Ứng dụng IoT — MUL14154  
**Trường:** Học viện Công nghệ Bưu chính Viễn thông (PTIT)  
**Lớp:** D22CQPTUD01-N  
**Nhóm:**  
- Phạm Nguyễn Hải Triều — N22DCPT104
- Nguyễn Mạnh Trí — N22DCPT103

---

## 📋 Mô Tả Đề Tài

Hệ thống IoT liên tục nhận traffic mạng. Khi hacker thay đổi chiến thuật tấn công, model AI cũ sẽ dần suy giảm hiệu suất — đây gọi là **concept drift**. Đề tài xây dựng pipeline:

- Phân loại traffic IoT (bình thường / tấn công) bằng **LSTM**
- Phát hiện drift tự động bằng thuật toán **ADWIN** (Adaptive Windowing)
- Tự động retrain model khi phát hiện drift (**Adaptive Retraining**)
- Lưu trữ và quản lý model trên **Azure Blob Storage**

---

## 🗂️ Dataset

| Dataset | Nguồn | Mục đích | Cách mô phỏng drift |
|---------|-------|----------|---------------------|
| **CICIoT2023** | University of New Brunswick | Dataset chính | Pha 1: Benign + Recon → Pha 2: Benign + Mirai (loại tấn công mới) |
| **TON_IoT** | UNSW Sydney | Dataset xác nhận | Pha 1: Normal + Backdoor → Pha 2: Normal + DDoS; chia theo cột timestamp thực tế nếu có |

> **Ghi chú về temporal split:**
> - **CICIoT2023** không có cột timestamp → drift được mô phỏng thủ công bằng cách ghép 2 pha.
> - **TON_IoT** được sort theo cột `ts` nếu tìm thấy, nếu không giữ thứ tự gốc (capture order).
> Đây là cách xử lý **đúng theo phương pháp luận** — không sort dữ liệu không có thời gian thực.

---

## 🏗️ Kiến Trúc Hệ Thống

```
[IoT Traffic Stream]
        │
        ▼
[Tiền xử lý: RobustScaler + fillna(median) + clip 0.5–99.5%]
        │
        ▼
[LSTM Model — phân loại binary: bình thường / tấn công]
        │ error signal (đúng/sai mỗi sample)
        ▼
[ADWIN Detector — theo dõi error rate theo cửa sổ thích nghi]
        │
   Drift detected?
  YES ──────────── Retrain model trên buffer 1000 mẫu gần nhất
   │                        │
   │               Upload weights lên Azure Blob Storage
   ▼
[Azure Blob Storage]
   ├── model-registry/   ← weights (.h5)
   └── results/          ← JSON số liệu + PNG biểu đồ
```

---

## 📊 Kết Quả Thực Nghiệm

> ⚠️ Các giá trị dưới đây là **kết quả mẫu** từ một lần chạy thực nghiệm.
> Kết quả sẽ thay đổi theo phiên chạy do tính ngẫu nhiên trong quá trình train.

### Đóng Góp 1 — Drift Detection Delay (CICIoT2023, δ so sánh)

| Delta (δ) | Phát hiện tại (sample) | Độ trễ (samples) | False Positive |
|-----------|------------------------|-------------------|----------------|
| 0.1       | ~25,439                | ~457              | 0              |
| 0.002     | ~25,855                | ~873              | 0              |
| 0.0001    | ~26,079                | ~1,097            | 0              |

→ δ lớn → phát hiện nhanh hơn nhưng nguy cơ báo nhầm cao hơn.
→ δ nhỏ → chắc chắn hơn, trễ hơn, phù hợp môi trường critical.

### Đóng Góp 2 — Static vs Adaptive (CICIoT2023)

| Giai đoạn | Static F1 | Adaptive F1 |
|-----------|-----------|-------------|
| Trước drift | ~0.90 | ~0.90 |
| Sau drift   | ~0.35 ↓ | ~0.78 ↑ |

→ Static model không phục hồi sau drift; Adaptive phục hồi sau ADWIN detect và retrain.

### Đóng Góp 3 — Trade-off Accuracy vs Update Cost

| Chiến lược cập nhật | F1 sau drift | Chi phí |
|---------------------|-------------|---------|
| Không cập nhật (Static) | ~0.35 | 0s |
| ADWIN-triggered (đề tài) | ~0.78 | ~2–5s/lần |
| Cập nhật liên tục (ước tính) | ~0.95 | >>100s |

→ Phương pháp ADWIN-triggered đạt điểm **tối ưu Pareto**: chi phí thấp, chất lượng cao.

---

## ☁️ Azure Blob Storage — Cấu Trúc

```
[Storage Account: iotdriftstore]
│
├── model-registry/                        (container)
│   ├── models/init_cic.weights.h5         ← weights CICIoT trước train
│   ├── models/init_ton.weights.h5         ← weights TON_IoT trước train
│   ├── models/final_adaptive_cic.weights.h5
│   └── models/final_adaptive_ton.weights.h5
│
└── results/                               (container)
    ├── all_results.png                    ← 5 biểu đồ tổng hợp
    ├── results_cic.json                   ← số liệu chi tiết CICIoT
    └── results_ton.json                   ← số liệu chi tiết TON_IoT
```

> **Lưu ý:** File `.h5` (model weights) **không được push lên GitHub** (nằm trong `.gitignore`) vì kích thước lớn. Tất cả weights được lưu trên **Azure Blob Storage**.

---

## 🚀 Hướng Dẫn Chạy Lại

### 1️⃣ Yêu cầu hệ thống

- Google Colab (khuyến nghị chọn **GPU T4** để tăng tốc train)
- Tài khoản Kaggle (miễn phí)
- Tài khoản GitHub
- Tài khoản Azure (có thể dùng free tier / student subscription)

---

### 2️⃣ Chuẩn bị: Lấy Kaggle API Token

Kaggle API token dùng để tự động tải dataset trong **Cell 4**.

1. Đăng nhập [kaggle.com](https://www.kaggle.com)
2. Click avatar góc trên phải → **Settings**
3. Kéo xuống mục **API** → click **Create New Token**
4. File `kaggle.json` sẽ tự download về máy với nội dung:
   ```json
   {"username":"your_username","key":"xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"}
   ```
5. Giữ file này, sẽ upload trong Cell 4.

---

### 3️⃣ Chuẩn bị: Tạo GitHub Personal Access Token

Token này dùng để clone/push repo từ Colab trong **Cell 0**.

1. Đăng nhập GitHub → click avatar → **Settings**
2. Kéo xuống → **Developer settings** (cuối sidebar trái)
3. Chọn **Personal access tokens** → **Tokens (classic)**
4. Click **Generate new token (classic)**
5. Điền:
   - **Note:** `Colab IoT Drift`
   - **Expiration:** 90 days (hoặc No expiration nếu muốn)
   - **Scopes:** tick `repo` (toàn bộ quyền repo)
6. Click **Generate token** → copy token ngay (chỉ hiện 1 lần)

> Token có dạng: `ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`

---

### 4️⃣ Chuẩn bị: Setup Azure Blob Storage

Azure dùng để lưu model weights và kết quả trong **Cell 3, 10, 12, 13, 14, 16**.

**Bước 1 — Tạo Storage Account:**
1. Vào [portal.azure.com](https://portal.azure.com)
2. Tìm **Storage accounts** → **Create**
3. Điền:
   - **Resource group:** Tạo mới hoặc chọn có sẵn
   - **Storage account name:** `iotdriftstore` (phải unique toàn cầu, đặt tên khác nếu bị trùng)
   - **Region:** Southeast Asia (gần Việt Nam)
   - **Performance:** Standard
   - **Redundancy:** LRS (Locally-redundant)
4. Click **Review + Create** → **Create**

**Bước 2 — Tạo 2 containers:**
1. Vào Storage Account vừa tạo → **Containers** (bên sidebar)
2. Click **+ Container** → tạo `model-registry` (Access level: Private)
3. Click **+ Container** → tạo `results` (Access level: Private)

**Bước 3 — Lấy Connection String:**
1. Trong Storage Account → **Access keys** (sidebar)
2. Click **Show** ở key1 hoặc key2
3. Copy **Connection string** (dạng `DefaultEndpointsProtocol=https;AccountName=...`)

---

### 5️⃣ Chuẩn bị: Tạo repo GitHub

1. Tạo repo mới trên GitHub (ví dụ tên: `iot-drift-detection`)
2. Init với README hoặc để trống đều được
3. Đảm bảo repo là **Public** (để tiện demo) hoặc Private tùy ý

---

### 6️⃣ Cài đặt Colab Secrets

Colab Secrets giúp lưu thông tin nhạy cảm mà không hardcode vào code. Dùng cho **Cell 0** và **Cell 3**.

1. Mở notebook trong Google Colab
2. Click biểu tượng **🔑 (chìa khóa)** ở sidebar trái
3. Click **+ Add new secret** và thêm lần lượt 4 secret sau:

| Tên Secret | Giá trị | Mô tả |
|------------|---------|-------|
| `GITHUB_TOKEN` | `ghp_xxxx...` | GitHub Personal Access Token (bước 3) |
| `GITHUB_USER` | `HaiTrieu186` | Username GitHub của bạn |
| `REPO_NAME` | `iot-drift-detection` | Tên repo GitHub (bước 5) |
| `AZURE_CONN_STR` | `DefaultEndpointsProtocol=https;...` | Azure Connection String (bước 4) |

4. Bật toggle **"Notebook access"** cho từng secret

> ⚠️ **KHÔNG** paste token hay connection string trực tiếp vào cell code — sẽ bị lộ khi push lên GitHub.

---

### 7️⃣ Thứ Tự Chạy Các Cell

Mở file `IoT_Drift_Detection_N22DCPT104_N22DCPT103.ipynb` trong Google Colab, chạy **theo đúng thứ tự** từ trên xuống:

| Cell | Tên | Mô tả | Thời gian ước tính |
|------|-----|-------|-------------------|
| **Cell 0** | Kết nối GitHub | Clone repo từ GitHub dùng token trong Secrets | ~15s |
| **Cell 1** | Cài thư viện | pip install TF, River, Azure SDK, Kaggle CLI... | ~3–5 phút |
| **Cell 2** | Import | Import tất cả thư viện, set seed | ~10s |
| **Cell 3** | Cấu hình Azure | Đọc `AZURE_CONN_STR` từ Secrets, test kết nối | ~5s |
| **Cell 4** | ⬆️ Upload kaggle.json + Tải dataset | **Popup yêu cầu upload file** → chọn `kaggle.json` từ máy tính; tự động tải 3 dataset từ Kaggle | ~15–25 phút |
| **Cell 5** | Liệt kê file | In cấu trúc folder và tên cột dataset | ~10s |
| **Cell 6** | Load CICIoT2023 | Ghép phase1 (Benign+Recon) + phase2 (Benign+Mirai) | ~1–2 phút |
| **Cell 7** | Load TON_IoT | Load + sort theo timestamp nếu có; ghép phase1/phase2 | ~30–60s |
| **Cell 8** | Tiền xử lý | RobustScaler, clip, fillna(median), reshape LSTM | ~1–2 phút |
| **Cell 9** | Build LSTM | Khởi tạo kiến trúc model | ~5s |
| **Cell 10** | ⚙️ Train offline | Train ban đầu 30 epochs + EarlyStopping + upload Azure | ~10–20 phút |
| **Cell 11** | Define hàm online | Định nghĩa `run_online()` — không có output | ~2s |
| **Cell 12** | 🔄 Online CICIoT | Chạy vòng lặp online learning trên CICIoT2023 | ~20–35 phút |
| **Cell 13** | 🔄 Online TON_IoT | Chạy vòng lặp online learning trên TON_IoT | ~15–25 phút |
| **Cell 14** | 📏 Đo Delay | Test ADWIN với 3 giá trị delta, in bảng độ trễ | ~15–20 phút |
| **Cell 15** | Số liệu báo cáo | Tổng hợp F1 static/adaptive, chi phí retrain | ~5s |
| **Cell 16** | 📊 Vẽ biểu đồ | Sinh 5 biểu đồ + upload PNG lên Azure | ~1–2 phút |
| **Cell 17** | Push GitHub | Tạo README.md + commit + push toàn bộ lên GitHub | ~30s |

> **Lưu ý Cell 4:** Khi cell chạy đến dòng `files.upload()`, một popup sẽ xuất hiện yêu cầu chọn file. Chọn file `kaggle.json` đã tải về ở bước 2. Sau đó cell sẽ tự động tải 3 dataset.

---

## 🛠️ Tech Stack

| Công nghệ | Phiên bản | Mục đích |
|-----------|-----------|----------|
| TensorFlow / Keras | 2.x | LSTM model |
| River | 0.21.2 | ADWIN drift detector |
| Azure Blob Storage SDK | latest | Cloud model storage |
| scikit-learn | latest | RobustScaler, F1, class_weight |
| pandas / numpy | 2.2.2 / 1.26.4 | Data processing |
| Kaggle CLI | latest | Tự động tải dataset |

---

## 📁 Cấu Trúc Repository

```
iot-drift-detection/
├── IoT_Drift_Detection_N22DCPT104_N22DCPT103.ipynb  ← notebook chính
├── README.md
├── all_results.png          ← 5 biểu đồ tổng hợp (auto-generated)
├── results_cic.json         ← số liệu CICIoT (auto-generated)
├── results_ton.json         ← số liệu TON_IoT (auto-generated)
└── .gitignore               ← chứa: kaggle.json, *.h5
```

> Model weights (`.h5`) **không nằm trong repo** — được lưu trên Azure Blob Storage.

---

## ❓ Xử Lý Lỗi Thường Gặp

**`AZURE_CONN_STR` không tìm thấy / Azure error:**
→ Kiểm tra lại Colab Secrets đã bật "Notebook access" chưa. Runtime → Restart runtime rồi chạy lại Cell 3.

**Cell 4 download lâu / timeout:**
→ Kaggle dataset ~2–4 GB tổng. Nếu timeout, chạy lại Cell 4, Kaggle CLI sẽ tiếp tục từ phần chưa tải (có cache).

**`ValueError: No objects to concatenate` ở Cell 6 hoặc 7:**
→ Folder dataset tên không khớp. Chạy lại Cell 5 để xem tên folder thực tế, điều chỉnh keyword trong `load_cic_folder()`.

**`best_val_accuracy < 0.6` ở Cell 10:**
→ ADWIN vẫn hoạt động nếu error rate thay đổi. Không cần dừng — xem phần "📊 Error TRƯỚC/SAU drift" để xác nhận drift signal đủ mạnh.

**Cell 12/13 không thấy `⚠️ DRIFT detected`:**
→ Thử chạy lại Cell 14 với δ=0.1 để kiểm tra — nếu vẫn không detect, khả năng model chưa học đủ tốt. Tăng số epochs ở Cell 10 lên 50.

---

## 📚 Tài Liệu Tham Khảo

- Bifet, A. & Gavaldà, R. (2007). *Learning from Time-Changing Data with Adaptive Windowing*. SIAM SDM 2007.
- Gama, J. et al. (2014). *A Survey on Concept Drift Adaptation*. ACM Computing Surveys, 46(4).
- Neto et al. (2023). *CICIoT2023: A Real-Time Dataset and Benchmark for Large-Scale Attacks in IoT Environments*. Sensors, 23(13).
- Moustafa, N. (2021). *TON_IoT: The Role of Heterogeneity and the Need for Standardisation of Heterogeneous Training Datasets*. ITNAC 2021.

---

*Đồ án môn Phát triển Ứng dụng IoT — MUL14154 | PTIT | 2024–2025*
