# 🔍 IoT Concept Drift Detection
> Phát hiện drift trong hệ thống IoT bằng Online Learning

**Môn học:** Phát triển Ứng dụng IoT
**Trường** Học viện Công nghệ Bưu chính viễn Thông (PTIT)
**Lớp** D22CQPTUD01-N
**Nhóm:**
- Phạm Nguyễn Hải Triều — N22DCPT104
- Nguyễn Mạnh Trí — N22DCPT103
**Kỹ thuật:** LSTM + ADWIN + Azure Blob Storage

---

## 📋 Mô Tả Đề Tài

Hệ thống IoT liên tục nhận traffic mạng. Khi hacker thay đổi chiến thuật tấn công,
model AI cũ sẽ dần suy giảm hiệu suất (concept drift). Đề tài xây dựng hệ thống:
- Phát hiện drift tự động bằng thuật toán ADWIN
- Tự động retrain model khi phát hiện drift
- Lưu trữ và quản lý model trên Azure Cloud

---

## 🗂️ Dataset

| Dataset | Nguồn | Mục đích | Drift được tạo như thế nào |
|---------|-------|----------|----------------------------|
| **CICIoT2023** | University of New Brunswick | Dataset chính | Pha1: Benign+Recon → Pha2: Benign+Mirai |
| **TON_IoT** | UNSW Sydney | Dataset xác nhận | Pha1: Normal+Backdoor → Pha2: Normal+DDoS |

---

## 🏗️ Kiến Trúc Hệ Thống

```
[IoT Traffic Stream]
        │
        ▼
[LSTM Model — phân loại bình thường / tấn công]
        │ error signal
        ▼
[ADWIN Detector — theo dõi error rate]
        │
   Drift detected?
  YES ──────────── Retrain model trên buffer 1000 mẫu gần nhất
   │                        │
   │               Upload weights lên Azure
   ▼
[Azure Blob Storage — model-registry + results]
```

---

## 📊 Kết Quả Thực Nghiệm

### Đóng Góp 1 — Drift Detection Delay

| Delta (δ) | Phát hiện tại | Độ trễ | False Positive |
|-----------|--------------|--------|----------------|
| 0.1 | ~25,439 | ~457 mẫu | 0 |
| 0.002 | ~25,855 | ~873 mẫu | 0 |
| 0.0001 | ~26,079 | ~1,097 mẫu | 0 |

### Đóng Góp 2 — Static vs Adaptive

- **Static model:** F1 giảm mạnh sau drift (không phục hồi)
- **Adaptive model:** F1 phục hồi sau khi ADWIN detect và retrain

### Đóng Góp 3 — Trade-off

- Chi phí mỗi lần retrain: ~2–5 giây
- ADWIN-triggered << Continuous retraining về chi phí
- Chất lượng tương đương hoặc tốt hơn Static

---

## ☁️ Azure Cloud Storage

```
model-registry/
  models/init_cic.weights.h5           ← weights ban đầu CICIoT
  models/init_ton.weights.h5           ← weights ban đầu TON_IoT
  models/final_adaptive_cic.weights.h5 ← sau online learning CICIoT
  models/final_adaptive_ton.weights.h5 ← sau online learning TON_IoT

results/
  all_results.png   ← 5 biểu đồ kết quả
  results_cic.json  ← số liệu CICIoT
  results_ton.json  ← số liệu TON_IoT
```

---

## 🚀 Hướng Dẫn Chạy Lại

### Yêu cầu
- Google Colab (GPU T4)
- File `kaggle.json` (tải từ Kaggle → Settings → Create New Token)
- Colab Secrets (🔑 biểu tượng chìa khóa bên trái):
  - `GITHUB_TOKEN` — GitHub Personal Access Token
  - `GITHUB_USER` — HaiTrieu186
  - `REPO_NAME` — tên repo
  - `AZURE_CONN_STR` — Azure connection string

### Thứ tự chạy

| Cell | Mô tả | Thời gian |
|------|--------|----------|
| Cell 0 | Kết nối GitHub | ~10s |
| Cell 1 | Cài thư viện | ~3 phút |
| Cell 2 | Import | ~5s |
| Cell 3 | Kết nối Azure | ~5s |
| Cell 4 | **Tải dataset từ Kaggle** | ~15-20 phút |
| Cell 5 | Liệt kê file | ~10s |
| Cell 6 | Load CICIoT2023 | ~1 phút |
| Cell 7 | Load TON_IoT | ~30s |
| Cell 8 | Preprocessing | ~1 phút |
| Cell 9 | Build model | ~5s |
| Cell 10 | **Train offline** | ~10-15 phút |
| Cell 11 | Define hàm online | ~2s |
| Cell 12 | **Online CICIoT** | ~20-30 phút |
| Cell 13 | Online TON_IoT | ~15 phút |
| Cell 14 | **Đo delay** | ~15 phút |
| Cell 15 | Số liệu báo cáo | ~5s |
| Cell 16 | Vẽ biểu đồ | ~1 phút |
| Cell 17 | Push GitHub | ~30s |

---

## 🛠️ Tech Stack

| Công nghệ | Phiên bản | Mục đích |
|-----------|-----------|----------|
| TensorFlow/Keras | 2.x | LSTM model |
| River | 0.21.2 | ADWIN drift detector |
| Azure Blob Storage | — | Cloud model storage |
| scikit-learn | latest | RobustScaler, metrics |
| pandas/numpy | 2.2.2 / 1.26.4 | Data processing |

---

## 📚 Tài Liệu Tham Khảo

- Bifet, A. & Gavaldà, R. (2007). *Learning from Time-Changing Data with Adaptive Windowing*. SDM 2007.
- Gama, J. et al. (2014). *A Survey on Concept Drift Adaptation*. ACM Computing Surveys.
- CICIoT2023: Neto et al. (2023). *CICIoT2023: A Real-Time Dataset and Benchmark for Large-Scale Attacks in IoT*.
- TON_IoT: Moustafa, N. (2021). *TON_IoT: The Role of Heterogeneity and the Need for Standardisation*.

---

*Đồ án môn Phát triển Ứng dụng IoT — MUL14154*
