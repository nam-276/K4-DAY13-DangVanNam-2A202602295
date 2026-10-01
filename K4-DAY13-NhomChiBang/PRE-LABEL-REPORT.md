# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: chưa được LC cấp trong buổi này.
- Thành viên: xem `TEAMMATES.md`.
- Trạng thái: `executed-by-group`.
- Người thực sự chạy: Nguyễn Chí Bằng, trên máy Mac của người đó. Ngày/giờ: 2026-10-01, docker-load bắt đầu 07:56:49 UTC (14:56 UTC+7), QC kết thúc 07:58:03 UTC. Hệ máy: macOS, Docker Engine 29.8.1, server Linux `arm64` (`aarch64`), 6 CPU, giới hạn container 4 CPU / 4 GB. `smoke.json` có `status: passed`.
- Image tag: `day13-pointpillars:lc-20261001-arm64`. Image ID: `sha256:dd6999ad5dd67962fdca18eb526132c08ce105980c9d89475d66cb1a5193f8a1`. Nạp từ `student-prelabel-arm64/image.tar.gz`, không pull registry. Repo revision trong manifest: `0831856d921609312d42c7582c366e5a311bb7b1` (`working_tree_dirty: true`).
- PCD: `input/demo.pcd` trong gói Student, frame_id `demo`, KITTI / MMDetection3D demo `000008`, 17238 điểm, sha256 `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`. Chạy local trên laptop, không phải máy LC.
- Checkpoint: `/opt/PointPillars/pretrained/epoch_160.pth`, sha256 `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`.
- Phạm vi: front-window (`rear=0` ở cả ba lượt). Score threshold: `0.3`.
- Kênh thứ tư: reflectance nguồn bị bỏ; adapter dùng kênh hằng, RGB=0 là placeholder, không phải intensity phục hồi. `z_ground` ước lượng từ PCD = `0.075` m. Phép z của script: `z_model = z_source - z_ground - delta`, rồi `z_source = z_model + z_ground + delta`. JSON đã ở hệ nguồn.

## Ba lượt inference thật

Cùng frame `demo`, dataset `KITTI`, checkpoint và score `0.3`. A→B chỉ đổi `delta`. B→C chỉ đổi pillar.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `run-A/boxes-demo-delta-0-voxel-0.16.json`, `side-demo-delta-0-voxel-0.16.png`, `summary.csv` | 1 `vehicles`, tâm x=13.154 y=-0.451 z=0.330, score=0.322. Ảnh Side: một hộp đỏ bám cụm điểm gần x≈13 m, đáy sát đường z=0. |
| B | 1.73 | 0.16 | 13 | 1.034 | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`, `side-demo-delta-1.73-voxel-0.16.png`, `summary.csv` | 10 `vehicles`, 2 `pedestrian`, 1 `two-wheels`. z từ 0.698 đến 1.426. Ảnh Side: nhiều hộp dọc theo cụm điểm, không phải một hộp của A được dịch đều. |
| C | 1.73 | 0.32 | 6 | 1.091 | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`, `side-demo-delta-1.73-voxel-0.32.png`, `summary.csv` | Cả 6 hộp đều `pedestrian`. Không còn `vehicles` hay `two-wheels`. mean_z gần B (1.091 so với 1.034) nhưng tập hộp khác. |

- A/B: thay input trước model không giống dịch cùng một hằng số cho output. A có 1 hộp, B có 13 hộp và thêm class. Chênh mean_z là 0.704 m, không phải 1.73 m. Hộp duy nhất của A (x=13.154, y=-0.451, z=0.330) không khớp một hộp B cùng x/y chỉ lệch z đúng 1.73 m. Model chạy lại trên đám mây đã đổi chiều cao trước inference, nên số hộp và danh tính hộp đổi.
- B/C: cùng delta 1.73, pillar 0.16 → 0.32 làm mất toàn bộ xe và xe hai bánh của B, chỉ còn 6 người đi bộ. Đây là đổi biểu diễn pillar đưa vào mạng, không phải quên cộng z ngược (mean_z gần nhau). Không đủ bằng chứng để chọn cấu hình tốt hơn: nhiều hộp hơn hoặc score cao hơn (B có xe score 0.933) không phải nhãn đúng. Không có ground truth của frame này trong bài.
- Giới hạn ROI: log ba lượt đều `rear=0`, nên vật phía sau không được tính là model bỏ sót khi so A/B/C. Ảnh Side là chiếu x–z toàn scene, các vật khác y có thể chồng lên nhau. Không dùng Side một mình để kết luận yaw hoặc hình học từng hộp; cần Top/Front và camera khi sửa frame Robotaxi.
- Không JSON nào đủ cơ sở để import vào job CVAT Robotaxi. Đây là frame KITTI `demo` `000008`, khác 30 frame nguồn. Prediction chỉ là gợi ý thí nghiệm. Việc cần kiểm tiếp trên frame thật là class, tâm, kích thước, hướng và hộp thiếu/thừa bằng PCD cùng ảnh camera, không lấy số hộp KITTI làm đáp án.

## Ca QC có kiểm soát — không import CVAT

Helper `pipeline-qc-cases.py` tạo ba ca từ prediction B (`boxes-demo-delta-1.73-voxel-0.16.json`, sha256 `51f49ac6ba8c97f8a2458e289fee48a33e1d07dd94de99ba935c142fd7e26016`). Không chạy lại model. `qc-cases/manifest.json`: `training_only: true`, `height_offset_m: 1.805` (= delta 1.73 + z_ground 0.075). Không phải nhãn đúng.

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 13 | 0 | Không | Giữ bản prediction B để đối chiếu. Không coi là cuboid đúng. | So với JSON B: không trường hình học nào đổi. `case-correct.json`. |
| case-batch-z | 13 / 13 | Mỗi hộp z giảm đúng 1.805 m | Không. Class, x, y, yaw, kích thước, score giữ nguyên. | Dừng sửa tay cả batch. Kiểm phép chuyển z (quên bước ngược), yêu cầu tạo lại prediction. | `case-batch-z.json` và `side-batch-z.png`: cả dải hộp nằm dưới đường z=0, tách khỏi cụm điểm. |
| case-one-box-z | 1 / 13 | Hộp index 0 (`vehicles`, x≈8.09, y≈1.21) giảm đúng 1.805 m (z 0.921 → −0.884). 12 hộp còn lại không đổi. | Không, ngoài z của hộp 0. | Không dừng cả pipeline. Kiểm riêng hộp đó bằng nhiều view. | `case-one-box-z.json` và `side-one-box-z.png`: một hộp chìm dưới z=0 gần x≈8 m, các hộp khác vẫn bám điểm như B. |

## Nhận xét cá nhân

### Nguyễn Chí Bằng — 2A202602248

Tôi vận hành `student-bundle.py run` trên Mac arm64. Ba lượt và QC đều `passed` trong `smoke.json`.

Quan sát A/B: `summary.csv` ghi A 1 hộp mean_z 0.330, B 13 hộp mean_z 1.034. Ảnh `run-A/side-demo-delta-0-voxel-0.16.png` chỉ một hộp; `run-B/side-demo-delta-1.73-voxel-0.16.png` có nhiều hộp bám cụm điểm. Đổi `delta` trước inference không dịch mọi hộp thêm 1.73 m.

Phép z: `z_model = z_source - 0.075 - delta`, rồi JSON trả về `z_source = z_model + 0.075 + delta`. Quên bước ngược trừ một lượng `1.805` m cho z. Chiều thuận đưa PCD về hệ model; chiều ngược trả hộp về hệ nguồn. Hai bước không thay nhau được bằng cách cộng 1.73 vào output sau khi model đã chạy.

Quyết định lỗi batch: `case-batch-z` lệch cùng 1.805 m ở 13/13 hộp, class/x/y/yaw giữ nguyên, nên dừng sửa từng hộp và kiểm pipeline. `case-one-box-z` chỉ hộp 0 lệch, nên kiểm đối tượng đó, không kết luận cả batch hỏng.

Chưa chắc: chưa có nhãn KITTI để nói B hay C đúng hơn. Side không đủ để chốt yaw. Hoàng Văn Đạt và Nguyễn Việt Tiến chưa ghi nhận xét trên bản chạy này.

### Đặng Văn Nam — 2A202602295

Tôi đọc lại kết quả đã chạy trong thư mục nhóm và phụ trách phần đối chiếu A/B/C, đặc biệt là sự khác nhau giữa thay `delta` và đổi kích thước pillar. Tôi không phải người trực tiếp chạy Docker trên máy Mac; phần này dựa trên các file output đã lưu và `smoke.json` đã `passed`.

Quan sát A/B: `run-A/summary.csv` ghi 1 hộp với mean_z 0.330, còn `run-B/summary.csv` ghi 13 hộp với mean_z 1.034. Hai ảnh `run-A/side-demo-delta-0-voxel-0.16.png` và `run-B/side-demo-delta-1.73-voxel-0.16.png` cũng khác rõ về số hộp, nên không thể hiểu việc đổi `delta` như dịch cùng một hộp lên/xuống một hằng số.

Phép z tôi hiểu là script đưa điểm từ hệ nguồn sang hệ model bằng `z_model = z_source - z_ground - delta`, sau đó phải cộng ngược `z_ground + delta` khi xuất JSON. Với lượt B, lượng cộng ngược là `1.805` m. Nếu quên bước này thì hộp bị thấp hơn đúng lượng đó, thể hiện trong ca `case-batch-z`.

Quyết định QC: với `case-batch-z`, 13/13 hộp đều lệch z cùng `1.805` m trong khi class, x, y, yaw giữ nguyên, nên phải dừng cả batch và kiểm pipeline/chuyển hệ tọa độ thay vì sửa tay từng hộp. Với `case-one-box-z`, chỉ hộp index 0 lệch nên cần kiểm riêng hộp đó bằng nhiều view, chưa đủ cơ sở kết luận toàn bộ batch sai.

Chưa chắc: chưa có ground truth cho frame KITTI demo nên chưa thể nói B tốt hơn C chỉ vì B có nhiều hộp hơn. Ảnh Side chỉ là chiếu x-z, không đủ để kết luận yaw hay class cuối cùng khi đưa sang dữ liệu Robotaxi.

### Hoàng Văn Đạt — 2A202602267

Chưa tự viết. Cùng yêu cầu như mục trên.

### Nguyễn Việt Tiến — 2A202602315

Chưa tự viết. Cùng yêu cầu như mục trên.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
