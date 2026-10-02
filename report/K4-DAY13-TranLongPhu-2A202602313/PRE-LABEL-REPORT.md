# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## 1. Nhóm và provenance

- Mã nhóm/phòng: `K4-DAY13-TranLongPhu-2A202602313` (Bài làm cá nhân)
- Thành viên: xem `TEAMMATES.md` (Họ và tên: **Trần Long Phú**, MSSV: **2A202602313**, độc lập đảm nhiệm trọn vẹn tất cả các vai trò).
- Trạng thái: `executed-by-group` (học viên tự chạy trực tiếp trên máy bằng Docker engine local qua gói offline bundle `student-bundle.py`).
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Trần Long Phú (`phu`); 2026-10-02 09:37:12 UTC+7; Ubuntu 24.04 LTS (kernel 7.0.0-34-generic), x86_64 / amd64.
- Image tag và image ID; phiên bản repo: `day13-pointpillars:lc-20261001-amd64` (ID: `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`); repo revision: `0831856d921609312d42c7582c366e5a311bb7b1`.
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: `input/demo.pcd` (frame `demo`, 17238 points từ mẫu KITTI 000008); sha256: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`; chạy offline container local.
- Checkpoint: PointPillars KITTI có sẵn trong image: `/opt/PointPillars/pretrained/epoch_160.pth` (sha256: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`).
- Các cấu hình được giữ chung:
  - Phạm vi: `front-window` (chỉ cửa sổ phía trước của checkpoint KITTI: $x \in [0, 70.4]\text{ m}$, $y \in [-40, 40]\text{ m}$, $z \in [-3, 1]\text{ m}$).
  - Score threshold: `0.3` (ngưỡng tin cậy giữ cố định ở cả ba lượt).
  - Giả định kênh thứ tư/intensity: Lược bỏ reflectance gốc của KITTI, dùng adapter hằng số RGB=0.
  - Nguồn $z_{ground}$: `0.075 m` (ước lượng từ độ cao điểm thấp nhất của mặt đường trong PCD).

## 2. Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát. Số hộp không phải đáp án cần khớp nhóm khác.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| :---: | :---: | :---: | :---: | :---: | :--- | :--- |
| **A** | 0 m | 0.16 m | 1 | 0.330 m | `run-A/summary.csv`<br>`boxes-demo-delta-0-voxel-0.16.json`<br>`side-demo-delta-0-voxel-0.16.png` | Chỉ phát hiện duy nhất 1 hộp `vehicles` tại $(x=13.15, y=-0.45, z=0.33)\text{ m}$ với score thấp ($0.322$). Do delta=0m (chưa bù độ cao đặt cảm biến $1.73\text{ m}$ của KITTI), các điểm bị dịch xuống quá sâu so với mặt đất trong hệ tọa độ chuẩn của model -> model bỏ sót hầu hết các xe. |
| **B** | 1.73 m | 0.16 m | 13 | 1.034 m | `run-B/summary.csv`<br>`boxes-demo-delta-1.73-voxel-0.16.json`<br>`side-demo-delta-1.73-voxel-0.16.png` | Phát hiện 13 hộp (10 vehicles, 2 pedestrians, 1 two-wheels) với độ tin cậy rất cao (nhiều xe đạt score $> 0.85 - 0.93$). Khi bù delta=$1.73\text{ m}$, đám mây điểm khớp đúng hệ tọa độ mà checkpoint KITTI được huấn luyện, bao quát tốt các xe trên làn đường. |
| **C** | 1.73 m | 0.32 m | 6 | 1.091 m | `run-C/summary.csv`<br>`boxes-demo-delta-1.73-voxel-0.32.json`<br>`side-demo-delta-1.73-voxel-0.32.png` | Chỉ phát hiện 6 hộp và 100% bị gán nhãn `pedestrian` (0 vehicles). Tăng kích thước pillar lên $0.32\text{ m}$ làm diện tích ô tăng gấp 4 lần, làm thô đặc trưng không gian khiến model pretrained ở voxel $0.16\text{ m}$ mất khả năng nhận diện hình học xe lớn. |

### Phân tích chi tiết:
- **A/B — chỉ đổi delta**: A có 1 hộp; B có 13 hộp. Ảnh `side-demo-delta-1.73-voxel-0.16.png` và file JSON cho thấy ở dải $x \in [6, 20]\text{ m}$ có rất nhiều cụm xe được đóng hộp chính xác. Đây là chạy lại model trên input khác (được bù cao độ sensor trước khi qua PFN), không chỉ dịch hộp cũ sau inference. *Điều chưa chắc*: Số lượng hộp nhiều hơn ở B chưa hẳn đã hoàn hảo 100%, vẫn cần đối chiếu thêm góc nhìn Top/Front và ảnh camera để loại trừ false positives.
- **B/C — chỉ đổi pillar XY**: B có 13 hộp; C có 6 hộp. Ảnh `side-demo-delta-1.73-voxel-0.32.png` khác ở chỗ không còn hộp vehicle nào, chỉ còn các hộp nhỏ gán nhãn pedestrian. Số lượng/lớp/vị trí thay đổi như sau: mất hoàn toàn 10 xe lớn, sinh ra 6 hộp người đi bộ rải rác. *Có đủ bằng chứng để kết luận tốt hơn không?* Không, vì checkpoint dùng cố định được train cho voxel size $0.16\text{ m}$; việc tăng voxel lên $0.32\text{ m}$ làm hỏng độ phân giải biểu diễn đặc trưng không gian của mạng hiện tại.
- **Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw**: Góc Side là hình chiếu trên mặt phẳng $x\text{-}z$ (bị triệt tiêu trục $y$), làm các vật thể ở các làn đường khác nhau (khác tọa độ $y$) bị chiếu chồng lên nhau, không thể xác định hướng quay (yaw) hay chiều rộng (width) nếu chỉ dựa vào ảnh Side. Cần kết hợp Top-down (BEV), Front-view và Camera.
- **JSON nào còn chưa đủ cơ sở để import?** Cả ba JSON (A, B, C) đều chưa đủ cơ sở để import vào CVAT Robotaxi vì đây là kết quả thử nghiệm trên dữ liệu mẫu KITTI (khác domain cảm biến, khác góc đặt và khác frame với job bài tập Robotaxi). Cần nạp prediction Robotaxi do LC chạy trước trên portal.

## 3. Phép đổi z thuận/ngược trong bài

Trong pipeline KITTI của script:
- **Phép thuận (trước khi đưa vào model)**:
  $$z_{model} = z_{source} - z_{ground} - \delta$$
  Trong đó $z_{source}$ là cao độ trong hệ tọa độ LiDAR nguồn, $z_{ground} \approx 0.075\text{ m}$ là mặt đất cục bộ ước lượng từ PCD, và $\delta = 1.73\text{ m}$ là chiều cao sensor so với mặt đất. Phép trừ này chuẩn hóa đưa đám mây điểm về hệ quy chiếu mặt đất sensor của checkpoint KITTI (nơi mặt đường ở cao độ xấp xỉ $z = 0$).
- **Phép ngược (sau khi model dự đoán tâm hộp)**:
  $$z_{source} = z_{model} + z_{ground} + \delta$$
  Phép cộng này đưa tâm hộp 3D dự đoán từ hệ quy chiếu của model quay trở lại đúng hệ tọa độ ban đầu của file PCD nguồn.

### Vì sao đổi delta trước inference khác với dịch hộp sau inference?
- Đổi $\delta$ trước inference tác động trực tiếp vào các điểm mây LiDAR đưa vào mạng nơ-ron. Các lớp Pillar Feature Net (PFN) gom điểm vào các cột đứng và trích xuất đặc trưng hình học 3D dựa trên tọa độ điểm so với các anchor boxes đặt sẵn. Nếu delta sai, các điểm rơi ra ngoài vùng kích hoạt của anchor, khiến mạng không nhận diện được vật thể (như ở lượt A chỉ tìm được 1 xe thay vì 13 xe).
- Dịch hộp sau inference chỉ đơn thuần là phép cộng số học tịnh tiến tọa độ các hộp đã được phát hiện, hoàn toàn không thể giúp model phát hiện thêm các vật thể bị bỏ sót hay thay đổi confidence score.

## 4. Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| :---: | :---: | :---: | :---: | :---: | :--- |
| **case-correct** | 0 / 13 | 0 m | Không đổi | Kiểm tra kỹ từng đối tượng | Giữ nguyên 13 hộp của B, các hộp ôm khớp các cụm điểm trên `side-correct.png`. |
| **case-batch-z** | 13 / 13 | -1.805 m | Không đổi | **Dừng batch**, báo LC kiểm tra pipeline | Toàn bộ 13 hộp đều bị chìm sâu xuống dưới lòng đất đúng $1.805\text{ m}$ trên `side-batch-z.png`, dấu hiệu pipeline quên phép cộng z ngược. |
| **case-one-box-z** | 1 / 13 | -1.805 m | Không đổi | **Kiểm từng hộp** (lỗi cục bộ đối tượng) | Chỉ hộp đầu tiên (xe tại $x=8.09\text{ m}, y=1.21\text{ m}$) bị chìm trên `side-one-box-z.png`, 12 hộp còn lại nằm đúng vị trí. |

*Lưu ý: Helper tạo biến đổi có chủ đích từ prediction của lượt B, không phải kết quả inference riêng hoặc nhãn đúng.*

## 5. Nhận xét cá nhân — Trần Long Phú (MSSV: 2A202602313)

- **Vai trò đã làm**: Độc lập thực hiện toàn bộ quy trình bài làm cá nhân: kiểm tra Docker engine và môi trường Linux amd64; vận hành script `student-bundle.py run`; đối chiếu kết quả JSON, CSV và ảnh Side của 3 lượt A, B, C; phân tích 3 ca lỗi QC nhân tạo để phân biệt lỗi pipeline với lỗi đối tượng cục bộ; hoàn thành báo cáo kỹ thuật và cấu trúc thư mục bài nộp.
- **Quan sát A/B/C**:
  - Giữa A và B: Đưa delta từ $0\text{ m}$ lên $1.73\text{ m}$ giúp số hộp tăng từ 1 lên 13, score cao nhất đạt $0.933$ ở hộp vehicle ($x=8.09\text{ m}, y=1.21\text{ m}$), khẳng định tầm quan trọng của việc chuẩn hóa cao độ sensor trước khi đưa vào model.
  - Giữa B và C: Tăng voxel size lên $0.32\text{ m}$ khiến model pretrained mất hoàn toàn khả năng phát hiện xe (từ 10 xe về 0 xe), chỉ phát hiện 6 pedestrian với score thấp hơn nhiều, cho thấy sự nhạy cảm của feature extractor với kích thước pillar.
- **Diễn giải phép z thuận/ngược**: Phép thuận ($z_{model} = z_{source} - z_{ground} - \delta$) đưa điểm về hệ quy chiếu mặt đất của model. Phép ngược ($z_{source} = z_{model} + z_{ground} + \delta$) đưa kết quả dự đoán trả về hệ tọa độ gốc của cảm biến LiDAR.
- **Quyết định lỗi batch và hành động**:
  - Khi toàn bộ hộp trong frame đều bị lệch cùng một khoảng cao độ $z$ (như trong `case-batch-z` lệch đúng $1.805\text{ m}$): Đây là lỗi hệ thống trong pipeline chuyển đổi (quên phép cộng z ngược) -> **Hành động: Dừng sửa thủ công ngay lập tức**, báo LC để sửa code transform, không cố gắng tịnh tiến sửa tay từng hộp.
  - Khi chỉ 1 hộp bị lệch còn các hộp khác chuẩn (như `case-one-box-z`): Đây là lỗi cục bộ của model trên đối tượng cụ thể -> **Hành động: Kiểm tra từng hộp** và chỉnh sửa tay trên CVAT bằng nhiều góc nhìn.
- **Điều chưa chắc chắn**: Dữ liệu KITTI demo đã bị lược bỏ reflectance thật (dùng hằng số RGB=0); ảnh Side x-z bị chiếu chồng (bỏ qua trục y); kết quả KITTI không phản ánh trực tiếp độ chính xác trên dữ liệu xe Robotaxi thật tại Việt Nam.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
