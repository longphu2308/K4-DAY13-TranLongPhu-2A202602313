# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: Bài làm cá nhân (Học viên tự thực hiện)
- Thành viên: xem `TEAMMATES.md` (Họ và tên: Trần Long Phú, MSSV: 2A202602313, độc lập đảm nhiệm các vai trò).
- Trạng thái: `executed-by-group` (học viên tự chạy trực tiếp trên máy bằng gói offline bundle `student-bundle.py`).
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Trần Long Phú (phu); 2026-10-02 09:37:12 UTC+7; Ubuntu 24.04 LTS (x86_64 / amd64).
- Image tag và image ID; phiên bản repo: `day13-pointpillars:lc-20261001-amd64` (ID: `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`); repo revision: `0831856d921609312d42c7582c366e5a311bb7b1`.
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: `input/demo.pcd` (frame `demo`, 17238 points); sha256: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`; chạy offline container local.
- Checkpoint: PointPillars KITTI có sẵn trong image: `/opt/PointPillars/pretrained/epoch_160.pth` (sha256: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`).
- Phạm vi: front-window; score threshold: 0.3
- Giả định kênh thứ tư/intensity và nguồn z_ground: Lược bỏ reflectance gốc, dùng adapter hằng số RGB=0; `z_ground = 0.075 m` (ước lượng từ độ cao điểm thấp nhất của mặt đường trong PCD).

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát. Số hộp không phải đáp án cần khớp nhóm khác.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `run-A/summary.csv`, `boxes-demo-delta-0-voxel-0.16.json`, `side-demo-delta-0-voxel-0.16.png` | Chỉ phát hiện 1 hộp `vehicles` duy nhất tại x=13.15m, z=0.33m với score thấp (0.322). Do delta=0m (chưa bù 1.73m cao độ sensor KITTI), các điểm bị dịch xuống quá thấp so với mặt đất trong hệ model -> model bỏ sót hầu hết các xe. |
| B | 1.73 | 0.16 | 13 | 1.034 | `run-B/summary.csv`, `boxes-demo-delta-1.73-voxel-0.16.json`, `side-demo-delta-1.73-voxel-0.16.png` | Phát hiện 13 hộp (10 vehicles, 2 pedestrians, 1 two-wheels) với độ tin cậy cao (nhiều xe đạt score > 0.85 - 0.93). Khi bù delta=1.73m, đám mây điểm khớp hệ tọa độ chuẩn mà checkpoint KITTI được huấn luyện. |
| C | 1.73 | 0.32 | 6 | 1.091 | `run-C/summary.csv`, `boxes-demo-delta-1.73-voxel-0.32.json`, `side-demo-delta-1.73-voxel-0.32.png` | Chỉ phát hiện 6 hộp và 100% bị gán nhãn `pedestrian` (0 vehicles). Tăng kích thước pillar lên 0.32m làm diện tích ô tăng gấp 4 lần, làm thô đặc trưng không gian khiến model pretrained ở voxel 0.16m nhận diện sai hình học xe. |

- A/B — chỉ đổi delta: A có 1 hộp; B có 13 hộp. Ảnh `side-demo-delta-1.73-voxel-0.16.png` và file JSON khác ở dải x từ 6m đến 20m có rất nhiều cụm xe được đóng hộp chính xác. Đây là chạy lại model trên input khác (được bù cao độ sensor), không chỉ dịch hộp cũ; điều em còn chưa chắc là số lượng hộp nhiều hơn ở B chưa hẳn là đúng 100%, vẫn cần đối chiếu thêm góc nhìn Top/Front và ảnh camera để loại trừ false positives.
- B/C — chỉ đổi pillar: B có 13 hộp; C có 6 hộp. Ảnh `side-demo-delta-1.73-voxel-0.32.png` khác ở chỗ không còn hộp vehicle nào, chỉ còn các hộp nhỏ gán nhãn pedestrian. Số lượng/lớp/vị trí thay đổi như sau: mất hoàn toàn 10 xe lớn, sinh ra 6 hộp người đi bộ rải rác. Có đủ bằng chứng để kết luận tốt hơn không? Không, vì checkpoint dùng cố định được train cho voxel size 0.16m; việc tăng voxel lên 0.32m làm hỏng độ phân giải biểu diễn đặc trưng của mạng hiện tại.
- Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào? Góc Side là hình chiếu trên mặt phẳng x-z (bị triệt tiêu trục y), làm các vật thể ở các làn đường khác nhau (khác y) bị chiếu chồng lên nhau, không thể xác định hướng quay (yaw) hay chiều rộng (width) nếu chỉ dựa vào ảnh Side.
- JSON nào còn chưa đủ cơ sở để import? Cả ba JSON (A, B, C) đều chưa đủ cơ sở để import vào CVAT Robotaxi vì đây là kết quả thử nghiệm trên dữ liệu mẫu KITTI khác domain cảm biến, khác góc đặt và khác frame với job bài tập Robotaxi. Cần nạp prediction Robotaxi do LC chạy trước trên portal.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 13 | 0 m | Không đổi | Kiểm tra kỹ từng đối tượng | Giữ nguyên 13 hộp của B, các hộp ôm khớp các cụm điểm trên `side-correct.png`. |
| case-batch-z | 13 / 13 | -1.805 m | Không đổi | **Dừng batch**, báo LC kiểm tra pipeline | Toàn bộ 13 hộp đều bị chìm sâu xuống dưới lòng đất đúng 1.805m trên `side-batch-z.png`, dấu hiệu quên phép biến đổi ngược. |
| case-one-box-z | 1 / 13 | -1.805 m | Không đổi | **Kiểm từng hộp** (lỗi cục bộ đối tượng) | Chỉ hộp đầu tiên (xe tại x=8.09m, y=1.21m) bị chìm trên `side-one-box-z.png`, 12 hộp còn lại nằm đúng vị trí. |

Ghi rõ helper tạo biến đổi có chủ đích từ prediction, không phải kết quả inference riêng hoặc nhãn đúng.

## Nhận xét cá nhân — Trần Long Phú (MSSV: 2A202602313)

- **Vai trò đã làm**: Độc lập thực hiện toàn bộ quy trình cá nhân: kiểm tra Docker engine và môi trường Linux amd64; vận hành script `student-bundle.py run`; đối chiếu kết quả JSON, CSV và ảnh Side của 3 lượt A, B, C; kiểm tra 3 ca lỗi QC nhân tạo để phân biệt lỗi pipeline với lỗi đối tượng cục bộ; hoàn thành báo cáo kỹ thuật.
- **Quan sát A/B/C**: Giữa A và B, việc đưa delta từ 0 lên 1.73m làm số hộp tăng từ 1 lên 13, score cao nhất đạt 0.933 ở hộp vehicle (x=8.09m), chứng minh tầm quan trọng của việc chuẩn hóa cao độ sensor đưa vào model. Giữa B và C, việc tăng voxel size lên 0.32m làm model pretrained mất hoàn toàn khả năng phát hiện xe (từ 10 xe về 0 xe), chỉ phát hiện 6 pedestrian với score thấp hơn nhiều.
- **Diễn giải phép z thuận/ngược**:
  - Phép thuận (trước khi đưa vào model): $z_{model} = z_{source} - z_{ground} - \delta$. Với $z_{ground} \approx 0.075\text{ m}$ là mặt đất cục bộ ước lượng từ PCD và $\delta = 1.73\text{ m}$ là chiều cao sensor so với mặt đất. Phép trừ này đưa điểm về hệ quy chiếu mặt đất sensor của checkpoint KITTI.
  - Phép ngược (sau khi model dự đoán tâm hộp): $z_{source} = z_{model} + z_{ground} + \delta$. Phép cộng này đưa tâm hộp 3D dự đoán quay trở lại đúng hệ tọa độ ban đầu của file PCD nguồn.
- **Quyết định lỗi batch và hành động**: Khi phát hiện toàn bộ hộp trong frame đều bị lệch cùng một khoảng cao độ z (như trong `case-batch-z` lệch đúng $1.805\text{ m}$), đây là lỗi hệ thống của pipeline (quên phép cộng z ngược) -> **hành động: dừng sửa thủ công ngay lập tức**, báo LC để sửa code transform. Ngược lại, nếu chỉ có 1 hộp bị lệch còn các hộp khác chuẩn (như `case-one-box-z`), đây là lỗi cục bộ của model trên đối tượng đó -> **hành động: kiểm tra từng hộp** và chỉnh sửa tay trên CVAT.
- **Điều chưa chắc chắn**: Dữ liệu KITTI demo đã bị lược bỏ reflectance thật (dùng hằng số RGB=0); ảnh Side x-z bị chiếu chồng (bỏ qua trục y); kết quả KITTI không phản ánh độ chính xác trên dữ liệu xe Robotaxi thật tại Việt Nam.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:

