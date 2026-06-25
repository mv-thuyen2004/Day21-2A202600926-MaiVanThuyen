# BÁO CÁO THỰC HÀNH MLOPS: TỪ THỰC NGHIỆM CỤC BỘ ĐẾN TRIỂN KHAI LIÊN TỤC

* **Khóa học**: AIInAction - VinUni
* **Bài Lab**: Day 21 - CI/CD cho AI Systems
* **Học viên thực hiện**: Mai Văn Thuyên
* **GitHub Repository**: [mv-thuyen2004/Day21-2A202600926-MaiVanThuyen](https://github.com/mv-thuyen2004/Day21-2A202600926-MaiVanThuyen)
* **IP REST API Máy chủ ảo**: `http://34.30.72.75:8000`
* **GCS Bucket**: `gs://mlops-wine-bucket-mvt`

---

## 1. Kết Quả Thực Nghiệm & Siêu Tham Số Chọn Lọc (Bước 1)

### Quá trình thực nghiệm
Chúng tôi đã thực hiện huấn luyện mô hình **RandomForestClassifier** trên tập dữ liệu **Wine Quality (Phase 1)** và ghi nhận 5 lần chạy với các tổ hợp siêu tham số khác nhau lên MLflow cục bộ. 

### So sánh hiệu năng các lần chạy:
- **Lần chạy 1 (Mặc định)**: `n_estimators: 100`, `max_depth: 5`, `min_samples_split: 2` $\rightarrow$ Accuracy: **0.5640** | F1: **0.5534**
- **Lần chạy 2**: `n_estimators: 50`, `max_depth: 3`, `min_samples_split: 5` $\rightarrow$ Accuracy: **0.5580** | F1: **0.5185**
- **Lần chạy 3**: `n_estimators: 200`, `max_depth: 15`, `min_samples_split: 2` $\rightarrow$ Accuracy: **0.6640** | F1: **0.6620**
- **Lần chạy 4**: `n_estimators: 300`, `max_depth: 25`, `min_samples_split: 2` $\rightarrow$ Accuracy: **0.6760** | F1: **0.6751**
- **Lần chạy 5 (Tối ưu)**: `n_estimators: 100`, `max_depth: 20`, `min_samples_split: 2` $\rightarrow$ Accuracy: **0.6840** | F1: **0.6829**

### Lý do lựa chọn bộ siêu tham số tối ưu:
Bộ siêu tham số ở **Lần chạy 5** (`n_estimators: 100`, `max_depth: 20`, `min_samples_split: 2`) mang lại hiệu năng cao nhất trên tập đánh giá (Accuracy: 0.6840). Việc tăng số lượng cây (`n_estimators`) và chiều sâu tối đa (`max_depth`) giúp mô hình nắm bắt tốt hơn các đặc trưng hóa học phức tạp của rượu vang mà không bị quá khớp quá mức.

---

## 2. Kết Quả Đối Sánh Hiệu Năng Sau Khi Thêm Dữ Liệu (Bước 3)

Khi mô phỏng việc bổ sung dữ liệu mới từ Phase 2 (tổng số mẫu tăng từ 2998 lên 5996), mô hình được tự động huấn luyện lại thông qua pipeline CI/CD và đạt được hiệu suất vượt trội:

| Chỉ số | Bước 2 (Dữ liệu Phase 1 - 2998 mẫu) | Bước 3 (Dữ liệu Phase 1 + 2 - 5996 mẫu) |
|---|---|---|
| **Accuracy (Độ chính xác)** | 0.6840 | **0.7580** *(Vượt cổng kiểm duyệt 0.70)* |
| **F1 Score** | 0.6829 | **0.7570** |

**Nhận xét**: Việc bổ sung thêm dữ liệu mới chất lượng cao giúp tăng đáng kể khả năng tổng quát hóa của mô hình RandomForest, cải thiện độ chính xác từ `68.4%` lên `75.8%`.

---

## 3. Khó Khăn Gặp Phải & Cách Giải Quyết

1. **Khó khăn 1**: Phiên bản `scikit-learn` 1.4.2 yêu cầu biên dịch C++ từ mã nguồn trên Windows với Python 3.13, gây lỗi cài đặt thư viện cục bộ.
   * *Giải pháp*: Bỏ găm cứng phiên bản (unpin) trong `requirements.txt` để `pip` tự nạp bản phân phối nhị phân mới nhất tương thích hoàn toàn với Python 3.13.
2. **Khó khăn 2**: Xác thực Google Cloud Storage bị lỗi trong GitHub Actions runner do tệp cấu hình DVC cục bộ tìm file xác thực ở thư mục gốc (`sa-key.json`).
   * *Giải pháp*: Thay đổi cấu hình pipeline GitHub Actions để xuất file secret `CLOUD_CREDENTIALS` thành `sa-key.json` trực tiếp tại thư mục làm việc gốc của Runner thay vì thư mục tạm `/tmp`.
3. **Khó khăn 3**: Lệnh Deploy bị lỗi `Handshake failed` khi kết nối SSH vào máy ảo.
   * *Giải pháp*: Sử dụng tính năng quản lý khóa SSH an toàn của Google Compute Engine Metadata để lưu trữ khóa Public, đảm bảo khóa SSH không bị Google Metadata Agent tự động xóa định kỳ.
4. **Khó khăn 4**: Endpoint `/health` kiểm tra chất lượng bị báo đỏ (Status Code 1) khi Deploy vì máy ảo cần thời gian để kéo mô hình mới từ GCS về trước khi Uvicorn khởi chạy hoàn tất.
   * *Giải pháp*: Thay thế thời gian ngủ cố định `sleep 5` bằng vòng lặp thử lại thông minh (retry loop) tối đa 30 giây để đảm bảo dịch vụ khởi động hoàn tất trước khi kiểm tra sức khỏe.
