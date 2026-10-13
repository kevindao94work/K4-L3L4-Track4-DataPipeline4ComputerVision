# Nhật ký vận hành Lab Day 9

## Checkpoint 0 — đã kiểm tra

- Sinh viên: Đào Gia Bảo; MSSV `2A202602793`; tài khoản GitHub `kevindao94work`.
- Repo đích đã xác nhận Public và thuộc tài khoản đang đăng nhập: https://github.com/kevindao94work/K4-L3L4-Track4-DataPipeline4ComputerVision.
- Docker 28.5.1, Compose v2.40.2; Docker Desktop được khởi động; RAM máy 16 GiB, Docker khoảng 7,65 GiB; ổ đĩa còn khoảng 301 GiB.
- Giải nén gói chuẩn `dataset/day09-data.zip`; `python3 scripts/check_data.py`: 6/6 SHA-256 khớp.
- `.env` cục bộ đặt đúng MSSV, không bật công tắc gây lỗi; không đưa cấu hình này lên Git.
- ZIP vốn được track trong starter; gỡ khỏi index, giữ bản cục bộ, thêm quy tắc ignore. Không sửa lịch sử cũ.
- Cả ba dịch vụ healthy; container nhận đúng MSSV. `scripts/lab.sh step all --reset` và `scripts/lab.sh verify`: 27/27; Gold 1.020 khung, phiên bản `4bb2a3037896`.
- Hash 3 file bảo vệ khớp; chưa sửa mã triển khai.

## Checkpoint 1 — Bronze

- `step 1 --reset`, `verify --module 1`: 5/5. A 3.021, B 3.234 dòng; 26 trip, 2 video.
- Ingest lại A không thêm dòng; `history bronze.raw_telemetry_a`: một WRITE version 0, 3.021 dòng.
- Offset A 0 ms, B 1.500 ms. Câu 1 đã ghi các khác biệt schema thực tế.

## Checkpoint 2 — Silver telemetry

- `step 2`, `exp break-gate`, `verify --module 2`: đạt 7/7 sau phục hồi.
- Mỗi drive cách ly 20 trùng, 30 thiếu tốc độ, 3 ngoài khoảng; sạch A 2.968, B 3.181.
- Gate bắt 6 dòng bằng in_range; sau khôi phục ERROR=0, WARN=1. B bắt đầu 14:30:00.009 UTC.

## Checkpoint 3 — Silver frames

- `step 3`, `test`, `exp idempotency`, `exp threshold`, `verify --module 3`: 6/6 và 9 unit test đạt.
- 26 trip, 1.246 khung; overwrite hai lần giữ fingerprint cb6118dbb0b8 và số dòng; append 60 → 120.
- dHash 0/2/6: A 60/149/363; B 8/52/244. Gap trung bình 66/73 ms, tối đa 2.884/3.112 ms.

## Checkpoint 4 — Gold

- `step 4`, hai `cli export-dataset`, `exp leakage`, `verify --module 4`: 7/7.
- Cùng version 4bb2a3037896; 680/186/154 khung, 18/4/4 trip; A/B giữ 436/584 khung.
- Chia frame làm 26/26 trip giao split và 26/26 giao train/test; chia trip có 0.
- Đọc manifest thật: silver.frames version 27 sau thí nghiệm retry, Bronze raw version 0, video_files 1. Không thử đổi MSSV.

## Checkpoint 5 — Warehouse và Airflow

- `step 5`, `verify --module 5`: 2/2; sáu mart, gallery 1.020 dòng.
- `day09_reset` success rồi `lab-normal` success (37 task instance); verify 27/27.
- Base36 MSSV modulo 26 = 25 → trip-09608; adaptation vì ví dụ numeric của GUIDE không áp dụng.
- Run lab-failure: 1 failed, 7 upstream_failed, 29 success. Clear duy nhất silver_frames[25] → 30 success/7 upstream_failed. Clear riêng 7 downstream bị chặn → 37 success.
- Retry không đổi 1.246 frame/fingerprint cb6118dbb0b8 hoặc dataset version 4bb2a3037896; verify 27/27.
- Ảnh thật lấy bằng Playwright/Chrome headless sau khi native CUA lỗi ScreenCaptureKit; đã xem Graph và log chứa đúng trip.
- Đã bỏ LAB_FAIL_ONCE, recreate Airflow, xác nhận MSSV gốc; không sửa mã đề.
