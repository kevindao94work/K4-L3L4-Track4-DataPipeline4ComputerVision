# REPORT — Lab Day 9

> Điền **bằng lời và số liệu của chính bạn** (lấy từ đầu ra của `scripts/lab.sh step ...` và từ `report/evidence/*.json`). Bài làm công khai trên GitHub; giảng viên đối chiếu với lịch sử commit và với bài của các bạn khác. Mỗi câu 3 đến 8 dòng là đủ.

| | |
|---|---|
| Họ và tên | Đào Gia Bảo |
| MSSV (đúng bằng `LAB_STUDENT_ID` trong `.env`) | `2A202602793` |
| GitHub | `kevindao94work` |
| Trip dùng cho bài tập sập và chạy lại (xem GUIDE, bước 5) | `trip-09608` |

## Kết quả cuối

- `dataset_version` của bạn: `4bb2a3037896`  (phải khớp `report/evidence/run_summary.json`)
- Số khung hình train / val / test của bạn: **680 / 186 / 154** (tổng 1.020)
- `report/verify.txt`: tạo bằng `python3 scripts/check_submission.py --save-verify` (phải có `27/27 checks passed`).
- Ảnh chụp trong `report/screenshots/`: `airflow_graph.png`, `minio_lake.png`, `metabase_dashboard.png`, và ảnh bài tập sập/chạy lại (log task đỏ thấy trip của bạn).

## Bước 1 · Bronze

### Câu 1
Liệt kê **ít nhất 4 khác biệt** giữa telemetry nguồn A và nguồn B mà bạn thấy ở bronze. Vì sao pipeline giữ **mỗi nguồn một bảng bronze** và để mọi cột ở dạng chữ, thay vì gộp và ép kiểu ngay?

Bronze có 3.021 dòng A và 3.234 dòng B; 26 trip, 2 bản ghi video có SHA-256 riêng; module 1 đạt 5/5.
Bốn khác biệt quan sát được: `ts_utc` ISO UTC / `timestamp_ms` epoch ms; `speed_mps` m/s / `spd_kmh` km/h;
`heading_deg` / `hdg`; `gas`, `brake` dạng tỷ lệ / `throttle_pct`, `brake_pct` dạng phần trăm.
Ngoài ra `lat`, `lon` đổi thành `latitude`, `longitude`; metadata B khai báo đồng hồ nhanh 1.500 ms, A là 0 ms.
Hai bảng thô giữ nguyên tên cột, chuỗi và ô rỗng để đối chiếu nguồn, phát hiện schema drift và chạy lại chuẩn hoá mà không mất thông tin do ép kiểu sớm.
Ingest A lần thứ hai báo `already present`; lịch sử `bronze.raw_telemetry_a` vẫn chỉ có WRITE version 0, 3.021 dòng, chứng minh nhận diện nội dung bằng hash.

## Bước 2 · Silver telemetry

### Câu 2
Pipeline đưa hai nguồn về một schema như thế nào (đơn vị, kiểu thời gian, **độ lệch đồng hồ**)? Bằng chứng nào trong kết quả bước 2 cho thấy đã đúng? Dán bảng quarantine của bạn (số dòng theo từng lý do, từng drive) và giải thích vì sao dòng lỗi vào `rejects` thay vì bị xoá.

Staging A ép timestamp ISO sang UTC và tốc độ sang double; B đổi `spd_kmh / 3,6`, đổi epoch ms bằng `to_timestamp`, trừ offset 1.500 ms và chia bàn đạp phần trăm cho 100.
B bắt đầu ở `2025-10-08 14:30:00.009 UTC`, lệch video chỉ 9 ms; module 2 đạt 7/7, không còn trùng `(drive_id, ts)` và mọi tốc độ trong 0–70 m/s.

| Drive | duplicate | speed_missing | speed_out_of_range | Tổng cách ly | Dòng sạch |
|---|---:|---:|---:|---:|---:|
| drive_a | 20 | 30 | 3 | 53 | 2968 |
| drive_b | 20 | 30 | 3 | 53 | 3181 |

Bảo toàn: 2.968 + 53 = 3.021; 3.181 + 53 = 3.234. Quarantine giữ dòng và lý do để kiểm toán/sửa nguồn; xoá im lặng sẽ che mất mức độ lỗi và không thể đối chiếu số nhận vào.

### Câu 3
Vì sao test `max_gap_ms` là `warn` còn `in_range` là `error`? Kết quả thí nghiệm `break-gate` của bạn (`break_gate.json`): test nào bắt được, bao nhiêu dòng, và điều đó cho biết gì về vai trò của "cổng chất lượng"?

Test `max_gap_ms` (ngưỡng 1.000 ms) dùng severity `warn`: mất tín hiệu cần được theo dõi, nhưng khung không đủ đồng bộ có thể được gắn cờ và loại ở Gold.
`in_range` là `error` vì tốc độ ngoài 0–70 m/s vi phạm hợp đồng dữ liệu và có thể làm sai nhãn huấn luyện.
Trong `break_gate.json`, nới luật lên 1.000 m/s khiến `in_range_silver_telemetry_speed_mps__70__0` bắt đúng 6 dòng; dbt báo PASS=5 WARN=1 ERROR=1 SKIP=0.
Sau khôi phục: PASS=6 WARN=1 ERROR=0 SKIP=0, verifier 7/7. Đây là bằng chứng test độc lập chặn dữ liệu sai dù bước làm sạch đã chấp nhận nó; WARN được giữ, không bị tắt để đạt kết quả.

## Bước 3 · Silver frames

### Câu 4
Dựa vào `idempotency.json`: số dòng trước và sau hai lần chạy lại, và khi ghi kiểu `append`. Vì sao Delta `overwrite` theo partition (một trip) an toàn hơn xoá rồi ghi lại, nhất là khi giữa chừng có lỗi?

`idempotency.json` chọn `trip-00003` có 60 khung: toàn bảng trước và sau hai lần chạy lại đều 1.246 dòng, fingerprint đều `cb6118dbb0b8` (`unchanged=true`).
Bảng thử ghi append hai lần có 120 dòng = 2 × 60: retry theo cách này sẽ nhân đôi dữ liệu.
Delta overwrite có predicate `trip_id` thay thế đúng partition trong một commit giao dịch, giữ nguyên các trip khác và cho người đọc thấy snapshot nhất quán.
Xoá rồi ghi là hai bước riêng: sập giữa hai bước có thể để lại partition rỗng; append khi retry có thể thêm bản sao. Đo đạc chứng minh retry không đổi nội dung; `_processed_at` được loại khỏi fingerprint vì là thời điểm xử lý.

### Câu 5
Khung hình **đen** và khung hình **tối** khác nhau thế nào, pipeline xử lý khác nhau ra sao, bạn có đồng ý không? Dựa vào `threshold.json`: nếu chọn ngưỡng 0, 2, 6 thì bao nhiêu khung bị coi là trùng ở mỗi drive? Bạn chọn ngưỡng nào, vì sao, và ngưỡng đó phụ thuộc vào tốc độ xe thế nào?

| dHash khác tối đa | số khung trùng, drive A | số khung trùng, drive B |
|---|---|---|
| 0 | 60 | 8 |
| 2 | 149 | 52 |
| 6 | 363 | 244 |

Có 1.246 khung từ 26 trip (A 602, B 644), mọi `frame_id` duy nhất; module 3 đạt 6/6 và 9/9 unit test.
A có 6 đen, 9 mờ, 4 tối, 149 gần trùng, 7 mất đồng bộ; B có 4 đen, 0 mờ, 61 tối, 52 gần trùng, 7 mất đồng bộ (các cờ có thể chồng nhau).
Trong cửa sổ thiếu sáng cấy vào B, 10/10 khung được gắn tối nhưng không đen: tôi đồng ý giữ khả năng dùng chúng, vì tối vẫn có cấu trúc; Gold còn xét mờ/trùng/đồng bộ, nên cờ tối không bảo đảm ảnh cuối cùng được giữ.
Chọn ngưỡng 2 theo mặc định: 0 bỏ sót biến thể gần giống, 6 loại nhiều khung có thể khác cảnh. Xe chậm thường có thay đổi hình nhỏ, nhưng bảng ngưỡng chưa chứng minh quan hệ nhân quả với tốc độ; cần phân tầng theo tốc độ để hiệu chỉnh.
ASOF nối event time với telemetry gần nhất trước đó; gap trung bình làm tròn A/B là 66/73 ms, tối đa 2.884/3.112 ms, cho thấy vẫn có đoạn mất tín hiệu.

## Bước 4 · Gold

### Câu 6
Dựa vào `leakage.json`: chia theo khung hình làm bao nhiêu trip bị chia cắt, so với chia theo trip? Vì sao điều này làm điểm kiểm thử đẹp một cách giả tạo?

Thí nghiệm `leakage.json` dùng 1.020 khung hợp lệ thuộc 26 trip. Chia từng frame ngẫu nhiên 70/15/15 khiến 26 trip xuất hiện ở nhiều split, và cả 26 trip có mặt đồng thời ở train lẫn test.
Gold chia theo trip có 0 trip bị chia cắt; từng drive đều có train, val, test, với tổng 18/4/4 trip tương ứng.
Các khung lân cận cùng đường, ánh sáng và vật thể rất giống nhau; khi chia frame, mô hình có thể ghi nhớ cảnh đã thấy, làm điểm test cao mà không phản ánh khả năng khái quát sang chuyến mới.
Đây là phép đo sự giao nhau của trip, chưa phải thí nghiệm huấn luyện đo mức tăng điểm; không suy ra một phần trăm cải thiện giả cụ thể.

### Câu 7
Vì sao `dataset_version` là **băm nội dung** chứ không phải ngày giờ hay số đếm? Hai lần chạy liên tiếp cho kết quả gì? Vì sao `dataset_version` của bạn **khác** bạn bè dù cùng dữ liệu (liên hệ MSSV)?

Hai lần `cli export-dataset` liên tiếp đều trả `4bb2a3037896`, 1.020 khung; train/val/test = 680/186/154 và SHA manifest parquet cùng `496d3785a82ea605ea40dc06591530a45808f1ec6e1e0998480f730df844f22f` tại checkpoint 4.
Mã export băm SHA-256 của các cặp `frame_id|split` đã sắp xếp, lấy 12 ký tự; vì vậy thời điểm chạy không làm đổi version khi danh sách/nhãn split không đổi.
Gold xếp trip trong từng drive bằng `md5(trip_id || LAB_STUDENT_ID)`; MSSV chính xác `2A202602793` là chuỗi, không cần ép số.
Đổi MSSV có thể đổi thứ tự/split/version, nhưng không bảo đảm mọi MSSV đều cho version khác trên mẫu hữu hạn. Tôi không thực hiện thử đổi MSSV; bằng chứng ở đây là hai lần export và kiểm tra SQL, không phải một kết quả thử nghiệm đổi ID.
Hash này nhận diện frame/split, không băm trực tiếp mọi pixel hoặc mọi nhãn telemetry; cần lineage và hash đầu vào để kiểm toán đầy đủ.

### Câu 8
Đọc phần `lineage` trong `manifest.json` của dataset bạn. Trả lời "dataset này được tạo từ dữ liệu nào?" bằng chính nội dung đó. Nếu một người hỏi 3 tháng sau, thông tin nào giúp tái tạo lại đúng dataset này?

Manifest của `4bb2a3037896` tại checkpoint 4 ghi lineage thực tế như bảng dưới; dữ liệu gốc có SHA-256:
- A: `636ba1a98dfc5ba665d1c935e1d2064e69f4df360ff9ffbd15795957d5e7e9aa`.
- B: `bfa62703f645ff77a8578ae3fc74a00881140d38a404d43c8e2af265bbee9fdc`.

| Bảng đầu vào | Delta version checkpoint 4 | Version sau clean replay cuối |
|---|---:|---:|
| bronze.raw_telemetry_a | 0 | 0 |
| bronze.raw_telemetry_b | 0 | 0 |
| bronze.trips_raw | 0 | 0 |
| bronze.video_files | 1 | 1 |
| silver.frames | 27 | 25 |
| silver.telemetry | 0 | 0 |
| silver.frames_flagged | 0 | 0 |
| silver.frame_telemetry | 0 | 0 |

Version Delta chỉ ra snapshot; hash video kiểm tra đúng byte nguồn; manifest frame/split và MSSV giúp tái tạo phân chia. Cần giữ gói nguồn, cấu hình, mã/phiên bản công cụ và Delta log cùng các data file, vì version riêng lẻ không khôi phục được dữ liệu đã xoá. Reset sẽ tạo lại số version nội bộ; bảng trên ghi rõ hai thời điểm quan sát, còn dataset_version vẫn ổn định khi frame/split giữ nguyên. Manifest cuối được lưu nguyên nội dung trong `report/evidence/final_manifest.json` (bọc kèm MSSV).

## Bước 5 · Điều phối và phục vụ

### Câu 9
Mô tả chính xác điều xảy ra khi trip của bạn sập rồi chạy lại: bao nhiêu task chuyển đỏ, bao nhiêu task `upstream_failed`, bạn clear gì, dữ liệu có nhân đôi không, `dataset_version` có đổi không? Bằng chứng (ảnh, số) là gì?

Vì MSSV chứa chữ, ví dụ `int(MSSV)` trong GUIDE không chạy được. Tôi dùng thích nghi công khai `int('2A202602793', 36) = 8333563383089463`, modulo 26 = 25; danh sách trip thực tế đã sắp xếp chọn `trip-09608` (không đổi MSSV). Đây không phải công thức chính thức cho MSSV chữ của giảng viên.
DAG `lab-normal` từ lake trống thành công: 37 task instance, gồm 26 mapped frame task. Ở run `lab-failure`, đúng 1 task `silver_frames[25]` failed, 7 downstream `upstream_failed`, 29 task success.
Tôi clear đúng mapped task lỗi trước bằng API Python `clear_task_instances` của Airflow; nó chạy lại thành công, nhưng 7 downstream vẫn upstream_failed (30 success/7 blocked). Sau đó clear riêng 7 downstream này; không clear 25 partition đã thành công.
Kết quả cuối 37/37 success; verifier 27/27; 1.246 frame và fingerprint `cb6118dbb0b8` giống baseline, version vẫn `4bb2a3037896`, không nhân đôi dữ liệu. Warehouse có 6 mart, gallery 1.020 dòng bằng Gold.
Bằng chứng: `airflow_graph.png`, `airflow_failed_graph.png`, `airflow_failed_task_log.png` thấy đúng trip, `airflow_recovered_graph.png`; số trạng thái và phạm vi clear nằm trong `airflow_recovery.json`. Tôi đã bỏ LAB_FAIL_ONCE và xác nhận container giữ MSSV gốc.

### Câu 10
Nâng từ 2 drive lên **500 xe mỗi ngày**: ba thành phần nào của pipeline này sẽ vỡ đầu tiên, và bạn thay bằng gì (nối với slide 12, 22, 23)?

Lab thực tế chỉ có 2 drive, 26 trip, 1.246 khung lấy mẫu và 1.020 khung Gold; Docker cấp khoảng 7,65 GiB, Airflow LocalExecutor parallelism=2 và mapped extraction giới hạn 1 task/run. Dưới đây là dự báo thiết kế cho 500 xe/ngày, không phải benchmark 500 xe.
(1) FFmpeg/IO extraction: mỗi task tải lại toàn video drive và giải mã theo trip; tăng xe làm lặp tải/giải mã. Dùng worker phân tán theo đoạn video, cache bản nguồn gần worker, giải mã một lần rồi chia partition; đổi lại phải quản lý retry, tài nguyên và tính xác định.
(2) DuckDB/dbt + nạp Postgres một nút: nhiều tác vụ tranh chấp file cục bộ, RAM và thao tác thay bảng mart toàn phần. Dùng engine phân tán (Spark/Trino theo loại truy vấn), bảng lakehouse partition và mart cập nhật tăng dần; đổi lại tăng chi phí vận hành, cần kiểm tra lại ngữ nghĩa SQL và consistency.
(3) MinIO/đường truyền và nhiều object JPEG nhỏ: 1.246 ảnh cùng Delta log/manifest đã tạo nhiều request; 500 xe có thể bão hoà request, băng thông và metadata trước khi đầy dung lượng. Dùng object storage mở rộng, ingest theo batch, đóng shard dữ liệu huấn luyện và giữ manifest ánh xạ ảnh; đổi lại giảm khả năng truy cập ngẫu nhiên từng JPEG trực tiếp.
Theo các chủ đề ingestion, ELT và orchestration của slide 12/22/23, cần backpressure, hàng đợi/concurrency theo năng lực worker, giám sát latency/throughput và quality gate ở từng tầng. Chưa có slide đính kèm để đối chiếu nội dung từng số trang; các thay thế trên dựa vào topology/mã quan sát, không gán số đo giả cho slide.

## Câu hỏi riêng của bạn trên Metabase

Câu SQL bạn thêm vào dashboard và điều nó cho biết:

Dashboard thật: `http://localhost:3000/dashboard/2`; hai câu hỏi đã lưu và thực thi trạng thái `completed`, bằng chứng số trong `report/evidence/metabase_queries.json`.

**1. Bao: frame retention by drive (card 46)**

```sql
select drive_id, frames_total, frames_kept, round(100.0 * frames_kept / nullif(frames_total, 0), 2) as kept_pct
from marts.mart_frame_funnel
order by drive_id;
```

A giữ 436/602 = 72,43%; B giữ 584/644 = 90,68%. Chênh 18,25 điểm phần trăm giúp ưu tiên kiểm tra nguồn A; tỷ lệ giữ không tự chứng minh mô hình học tốt hơn, và không cộng các cờ loại vì chúng có thể chồng nhau.

**2. Bao: top 5 frame rejection trips (card 47)**

```sql
select drive_id, trip_id, frames, frames_kept,
       round(100.0 * (frames - frames_kept) / nullif(frames, 0), 2) as dropped_pct
from marts.mart_trip_summary
order by dropped_pct desc, trip_id
limit 5;
```

| Trip (đều drive_a) | Tổng frame | Giữ | Loại (%) |
|---|---:|---:|---:|
| trip-04622 | 60 | 26 | 56,67 |
| trip-03904 | 22 | 11 | 50,00 |
| trip-02086 | 60 | 32 | 46,67 |
| trip-08888 | 60 | 36 | 40,00 |
| trip-07774 | 34 | 21 | 38,24 |

`trip-04622` là điểm ưu tiên xem footage/quality flags; kết quả này đo hợp của các nguyên nhân loại qua `frames - frames_kept`, chưa xác định riêng nguyên nhân của từng trip.

## Nhìn lại

Điều bạn thấy bất ngờ nhất, điều gì còn chưa rõ, và một lỗi dữ liệu mà bạn nghĩ pipeline **chưa** bắt được (nếu có). Tối đa 8 dòng.

Bất ngờ nhất là B có 61 khung tối nhưng không có khung mờ được gắn cờ; tối không đồng nghĩa ảnh vô dụng, và tỷ lệ giữ B vẫn 90,68%.
Thí nghiệm retry làm rõ khác biệt giữa dữ liệu idempotent và trạng thái điều phối: partition đã sửa được, nhưng 7 downstream vẫn cần clear riêng.
Tôi chưa chắc ngưỡng dHash 2 tối ưu cho mọi tốc độ/ánh sáng; bảng ngưỡng cho thấy đánh đổi, chưa có nhãn thủ công để đo false positive.
Một lỗi có thể chưa được bắt là GPS nhảy vị trí hoặc camera bẩn nhưng vẫn đủ sáng/nét; đây là giả thuyết từ các luật hiện có, chưa được thử cấy lỗi.
Cần thêm kiểm tra tính liên tục không gian–thời gian và đánh giá chất lượng bằng mẫu có nhãn trước khi áp dụng ngoài lab.
