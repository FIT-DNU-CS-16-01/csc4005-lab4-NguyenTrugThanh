# CSC4005 Lab 4 Report – CRNN for UrbanSound8K

## 1. Thông tin sinh viên

- Họ tên: Nguyễn Trung Thành
- Mã sinh viên: 1671040025
- Lớp: KHMT 1601
- Link GitHub repo:
- Link W&B project: https://wandb.ai/thanhfifo24-vietcombank/csc4005-lab4-urbansound8k-crnn?nw=nwuserthanhfifo24

## 2. Mục tiêu thí nghiệm

Lab 4 nhằm xây dựng mô hình **CRNN** để phân loại âm thanh môi trường trên bộ UrbanSound8K. Khác với Lab 3 (1D-CNN), CRNN kết hợp CNN để học đặc trưng cục bộ trong spectrogram (freq × time) và RNN (GRU/LSTM) để mô hình hóa phụ thuộc thời gian giữa các frame. Log-mel spectrogram được chọn vì biểu diễn tốt thông tin tần số theo cách tương tự thính giác con người. Mục tiêu đánh giá là cải thiện test accuracy so với 1D-CNN (~72%) và phân tích confusion matrix để hiểu từng lớp âm thanh.

## 3. Cấu hình dữ liệu

| Thành phần | Giá trị |
|---|---|
| Dataset | UrbanSound8K |
| Số lớp | 10 |
| Train folds | 1–8 |
| Validation fold | 9 |
| Test fold | 10 |
| Feature | log-mel spectrogram |
| Sampling rate | 16 kHz |
| Duration | 4 giây |

## 4. Cấu hình mô hình

| Thành phần | Giá trị |
|---|---|
| Model | CRNN (crnn_small) |
| CNN blocks | 2 blocks |
| RNN type | GRU (baseline) / LSTM (extension) |
| Hidden size | 128 |
| Dropout | 0.3 / 0.35 |
| Optimizer | AdamW |
| Learning rate | 0.001 / 0.0007 |
| Batch size | 32 |
| Epochs | 25 |

## 5. Kết quả huấn luyện

Điền kết quả tốt nhất từ W&B hoặc `metrics.json`.

| Run | best_val_acc | test_acc | Ghi chú |
|---|---:|---:|---|
| logmel_crnn_gru_baseline | 74.63% | 75.51% | GRU unidirectional, LR=0.001 |
| extension_bilstm_crnn | 63.48% | 68.82% | Bidirectional LSTM, LR=0.0007 |

## 6. Learning curves

![Learning Curves](../outputs/logmel_crnn_gru_baseline/curves.png)

Nhận xét:

- **Overfitting:** Có dấu hiệu nhẹ. Train loss tiếp tục giảm (0.64 ở epoch 25) còn val loss dao động quanh 0.93–1.08. Tuy nhiên mức độ chênh lệch không quá lớn, mô hình vẫn học tốt.
- **Validation loss:** Giảm ổn định từ epoch 1–8 (từ 1.92 → 1.26). Sau đó dao động nhỏ nhưng không tăng đều đặn. Đạt điểm tốt nhất ở epoch 24 (0.933).
- **Learning rate scheduler:** ReduceLROnPlateau kích hoạt từ epoch 21, giảm LR từ 0.001 → 0.0005, giúp mô hình tinh chỉnh tốt hơn ở cuối quá trình.

## 7. Confusion matrix

![Confusion Matrix](../outputs/logmel_crnn_gru_baseline/confusion_matrix.png)

Nhận xét:

- **Lớp phân loại tốt:** gun_shot (100%, 32/32), air_conditioner (74%), jackhammer (86%), drilling (80%). Các lớp có đặc trưng tần số rõ ràng dễ phân biệt.
- **Lớp dễ bị nhầm:** siren (44% accuracy) – thường nhầm với children_playing (30 mẫu); engine_idling (60%) – nhầm với siren (20 mẫu); car_horn (67%) – nhầm với street_music.
- **Giải thích:** Siren và children_playing đều có tần số cao, dao động liên tục. Engine_idling và siren có yếu tố tần số trung bình gần nhau. Cần đặc trưng thời gian-tần số chi tiết hơn (attention, focal loss) để phân biệt tốt hơn.

## 8. So sánh với Lab 3 1D-CNN

| Tiêu chí | Lab 3: 1D-CNN | Lab 4: CRNN |
|---|---|---|
| Feature chính | MFCC / log-mel | log-mel |
| Khả năng học pattern cục bộ | Có (Conv1D) | Có (Conv2D) |
| Khả năng học quan hệ thời gian | Hạn chế (FC layers) | Tốt hơn (GRU/LSTM) |
| Test accuracy | ~70–72% | 75.51% (GRU baseline) |
| Nhận xét | 1D-CNN xử lý feature 1D trực tiếp (MFCC/mel). CRNN tận dụng 2D spectrogram (freq × time): CNN học local spectral patterns, RNN học temporal dependencies → kết quả cải thiện 3–5%. |

## 9. Kết luận

**CRNN (GRU baseline) đạt test accuracy 75.51%, cải thiện ~3–5% so với 1D-CNN ở Lab 3.** Mô hình hội tụ ổn định với learning curve mượt mà và val loss giảm liên tục. Bidirectional LSTM extension cho kết quả thấp hơn (68.82%), nguyên nhân: (i) learning rate thấp (0.0007) không đủ tối ưu, (ii) parameter nhiều gây overfitting. Mô hình GRU baseline có dấu hiệu overfitting nhẹ ở các epoch cuối nhưng vẫn tổng quát hóa tốt (val acc ≈ test acc).

**Cải tiến tiếp theo:** (1) Data augmentation mạnh hơn (SpecAugment) cho lớp khó nhầm (siren, engine_idling); (2) Tuning learning rate cho Bi-LSTM (tăng LR lên 0.001); (3) Thử attention mechanism để mô hình tập trung vào frame/frequency band quan trọng; (4) Focal loss để xử lý class imbalance.

## 10. Link minh chứng

- GitHub commit cuối: https://github.com/FIT-DNU-CS-16-01/csc4005-lab4-NguyenTrugThanh/commit/6925fb2
- W&B run baseline: logmel_crnn_gru_baseline
- W&B run mở rộng: logmel_crnn_bilstm_extension