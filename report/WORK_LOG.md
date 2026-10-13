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
