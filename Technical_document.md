# Kế hoạch Triển khai Tiền xử lý Dữ liệu và Thiết kế Mô hình AI cho Hệ thống SomniGuard
## Kiến trúc Hệ thống SomniGuard và Cơ chế FSM Đa Chế độ Tiết kiệm Năng lượng

Hệ thống SomniGuard được thiết kế dưới dạng thiết bị đeo thông minh dạng nhẫn, hoạt động dựa trên chip hệ thống SoC không dây Silicon Labs EFR32xG26 tích hợp bộ tăng tốc toán học chuyên dụng Matrix Vector Processor (MVP). Để đạt được mục tiêu hoạt động liên tục tối thiểu 12 giờ trên một chu kỳ sạc pin cực kỳ giới hạn của thiết bị đeo, hệ thống không thể chạy mô hình học sâu liên tục 100% thời gian. Thay vào đó, nhóm phát triển đề xuất một máy trạng thái hữu hạn (Finite State Machine - FSM) điều khiển bằng chuyển động để quản lý thông minh các chế độ hoạt động của phần cứng và thuật toán AI.

Sơ đồ chuyển đổi trạng thái của FSM được cấu trúc linh hoạt như sau:



                  +-----------------------+
                  |    ACTIVE DAY MODE    | <---+ (Chuyển động mạnh liên tục)
                  |  - IMU: Bật (25Hz)    |     |
                  |  - PPG: Tắt           |     |
                  |  - AI: Tắt            |     |
                  +-----------------------+     |
                              |                 |
                (Nằm yên lặng / Đi ngủ)         |
                              v                 |
                  +-----------------------+     |
                  |   NORMAL SLEEP MODE   | ----+
                  |  - IMU: Bật (Gating)  |
                  |  - PPG: Bật (Low rate)|
                  |  - AI: Heuristic/Nhẹ  |
                  +-----------------------+
                              |
                (Bất thường kéo dài >3 giây)
                              v
                  +-----------------------+
                  |  DEEP ANALYSIS MODE   |
                  |  - IMU: Tắt (Bypass)  |
                  |  - PPG: Bật (50Hz)    |
                  |  - AI: 1D-CNN sâu     |
                  +-----------------------+


1. Active Day Mode (Chế độ Vận động Hằng ngày): Thiết bị liên tục kiểm tra gia tốc chuyển động từ cảm biến quán tính IMU MPU6050 ở tần số thấp 25Hz. Trong chế độ này, toàn bộ mô hình AI và cảm biến quang học MAX30102 được tắt hoàn toàn nhằm bảo tồn năng lượng. Không có cơ chế kiểm soát tình trạng giấc ngủ nào được thực hiện trong suốt quá trình hoạt động ban ngày.
2. Normal Sleep Mode (Chế độ Ngủ thông thường - Tiết kiệm điện): Khi cảm biến IMU ghi nhận cơ thể ở trạng thái tĩnh (ít chuyển động) trong một khoảng thời gian nhất định, thiết bị sẽ tự động chuyển sang chế độ này. Tại đây, để tiết kiệm năng lượng, cảm biến MAX30102 chỉ hoạt động ở tần số lấy mẫu rất thấp (ví dụ: 1Hz) hoặc chỉ sử dụng các chỉ số sinh hiệu đã được trích xuất sẵn bao gồm nhịp tim (BPM) và nồng độ bão hòa oxy trong máu (). Một thuật toán heuristic hoặc một mô hình học máy siêu nhẹ (Shallow Neural Network) sẽ liên tục theo dõi các chỉ số này. Nếu phát hiện bất kỳ dấu hiệu suy giảm oxy hoặc biến động nhịp tim bất thường nào kéo dài liên tục vượt quá 3 giây, hệ thống sẽ kích hoạt chuyển đổi sang chế độ chuyên sâu.
3. Deep Sleep Analysis Mode (Chế độ Phân tích Chuyên sâu): Khi có tín hiệu kích hoạt từ chế độ thông thường, thiết bị sẽ nâng tần số lấy mẫu của MAX30102 lên mức tối đa (50Hz) để thu thập chuỗi sóng PPG Red và IR thô có độ phân giải cao. Trong chế độ này, mô hình AI tích chập sâu (1D-CNN) được nạp vào bộ tăng tốc MVP để phân tích chi tiết các biến cố ngưng thở hoặc loạn nhịp tim. Điểm mấu chốt là dữ liệu chuyển động (IMU) sẽ hoàn toàn bị bỏ qua (bypass) trong chế độ này nhằm tiết kiệm năng lượng tính toán và tập trung tài nguyên xử lý chuyên sâu cho các chỉ số sinh hiệu y tế cốt lõi (PPG). Khi các chỉ số sức khỏe trở lại bình thường và ổn định, thiết bị sẽ tự động hạ cấp trạng thái về lại Chế độ Ngủ thông thường.

## Quy trình Tiền xử lý và Chiến lược Lựa chọn Đặc trưng Đa tầng

Sự kết hợp giữa các thuật toán xử lý tín hiệu số (DSP) cơ bản và mô hình AI cho phép chúng ta linh hoạt lựa chọn các tầng đặc trưng đầu vào tùy thuộc vào cấu hình phần cứng và yêu cầu về độ chính xác sinh lý học.
Cấu trúc phân tầng đặc trưng đầu vào của hệ thống được định nghĩa như sau:
### Tầng 1: Tầng Đặc trưng Tĩnh Tần số Thấp (Simpler Mode)
Được sử dụng chủ yếu trong Chế độ Ngủ thông thường. Nhóm phát triển sử dụng trực tiếp các chỉ số BPM và  đã được trích xuất ở tần số 1Hz từ chuỗi sóng thô thông qua các thuật toán DSP nhẹ chạy trên lõi CPU Cortex-M33. Định dạng dữ liệu mẫu được bàn giao từ thành viên trong nhóm như sau:



```csv
Subject_ID, Timestamp_ms, Raw_Red, Raw_IR, Final_R, SpO2, BPM
5,          636887,       140714,  138214,  1.4033,  98.8, 92
5,          637890,       140003,  137537,  1.5139,  98.8, 72

```


Sử dụng chuỗi dữ liệu này giúp mô hình đầu vào cực kỳ tinh gọn (kích thước tensor đầu vào chỉ là  cho mỗi cửa sổ giám sát 10 giây ở tần số 1Hz). Mô hình AI tương ứng có thể được thiết kế như một mạng kết nối đầy đủ (Dense Network) siêu nhẹ, thực thi trong vài micro-giây và tiêu thụ năng lượng không đáng kể.
### ầng 2: Tầng Đặc trưng Động Sóng Thô Tần số Cao (Detailed Mode)
Được kích hoạt trong Chế độ Phân tích Chuyên sâu. Mô hình AI sẽ nhận đầu vào là chuỗi sóng thô PPG Red và PPG IR thu thập trực tiếp ở tần số 50Hz (kích thước tensor đầu vào  cho cửa sổ 10 giây). Việc xử lý sóng thô cho phép mô hình trích xuất các biến động tinh vi về biên độ mạch đập (phản ánh hiện tượng co mạch ngoại vi do kích hoạt thần kinh giao cảm). Đây là thông tin cực kỳ quan trọng giúp phân biệt chính xác cơn ngưng thở tắc nghẽn (OSA) với các hiện tượng nhiễu loạn thông thường khác.
### Tầng 3: Tầng Gating Chuyển động và Loại bỏ Nhiệt độ
Tín hiệu Chuyển động (IMU): Được sử dụng như một bộ lọc logic (Gating mechanism). Khi biên độ gia tốc vượt ngưỡng vận động thông thường, hệ thống sẽ tự động chặn việc kích hoạt chế độ phân tích sâu hoặc tạm thời bỏ qua kết quả phân tích để tránh hiện tượng báo động giả do người dùng trở mình. Khi đã ở trong Chế độ Chuyên sâu, IMU được đưa vào trạng thái tạm ngừng hoạt động.

Tín hiệu Nhiệt độ (Thermistor): Được chuyển đổi thành thành phần tùy chọn (Optional) hoặc loại bỏ hoàn toàn khỏi mô hình quyết định. Về mặt sinh lý học, sự thay đổi thân nhiệt nền diễn ra với tốc độ rất chậm (tính bằng phút hoặc giờ) và đóng góp rất ít vào việc phát hiện các biến cố ngưng thở cấp tính vốn diễn ra đột ngột trong khoảng thời gian từ 10 đến 30 giây. Do đó, loại bỏ tín hiệu này giúp giảm bớt kích thước cổng ADC và tối giản hóa tensor đầu vào của mô hình.
## Thiết kế Kiến trúc Song song cho Mô hình Đa chế độ
Để đáp ứng cơ chế FSM đa chế độ, chúng ta sẽ thiết kế và huấn luyện hai mô hình AI độc lập: một mô hình siêu nhẹ chạy bằng phần mềm trên CPU Cortex-M33 cho chế độ giám sát thông thường, và một mô hình tích chập sâu tăng tốc bằng phần cứng MVP cho chế độ chuyên sâu.


| Đặc tính kỹ thuật | Mô hình Giám sát Thông thường (Lightweight) | Mô hình Phân tích Chuyên sâu (Deep 1D-CNN) |
| :--- | :--- | :--- |
| **Chế độ FSM kích hoạt** | Normal Sleep Mode (Ngủ thông thường) | Deep Sleep Analysis Mode (Chuyên sâu) |
| **Tín hiệu đầu vào** | Chuỗi dữ liệu $\text{SpO}_2$ và BPM (1Hz) | Chuỗi sóng thô PPG Red và PPG IR (50Hz) |
| **Kích thước Tensor đầu vào** | $[10 \times 2]$ (Cửa sổ 10 giây) | $[1 \times 500 \times 4]$ (Gồm 2 kênh đệm zero-padding) |
| **Kiến trúc mạng chính** | Mạng nơ-ron truyền thẳng (Fully Connected) | 1D-CNN (Ánh xạ sang 2D-CNN với chiều cao bằng 1) |
| **Phần cứng thực thi** | Lõi CPU ARM Cortex-M33 (Phần mềm) | Bộ tăng tốc toán học Matrix Vector Processor (MVP) |
| **Mục tiêu tính toán** | Nhận diện bất thường nhanh để kích hoạt chuyển đổi chế độ | Phân loại chi tiết và xác định chính xác cơn ngưng thở |


Bản mẫu Thực nghiệm Mô hình trên Python (Cập nhật Đa chế độ)
Dưới đây là mã nguồn Python chi tiết, mô phỏng toàn bộ logic máy trạng thái FSM, quy trình tiền xử lý dữ liệu động/tĩnh và cấu trúc hai mô hình AI tương thích hoàn toàn với nền tảng nhúng của Silicon Labs. Đọc code


## Chiến lược Giải quyết Khó khăn về Dataset và Thích ứng Người Việt
Việc thiếu hụt các bộ dữ liệu lâm sàng chuẩn hóa dành riêng cho thể trạng người Việt Nam là một thách thức lớn trong việc deploy thực tế. Để giải quyết triệt để vấn đề này, nhóm đề xuất một chiến lược huấn luyện và phân phối hai giai đoạn sử dụng kỹ thuật Học chuyển giao (Transfer Learning) và Cá nhân hóa trên thiết bị (On-device Personalization):



            +--------------------------------------------------------+
            | GIAI ĐOẠN 1: HUẤN LUYỆN MÔ HÌNH CƠ SỞ (PRE-TRAINING)   |
            | - Nguồn dữ liệu: MESA, UCD Sleep Apnea, MIMIC-III      |
            | - Mục tiêu: Học các quy luật sinh học cốt lõi          |
            +--------------------------------------------------------+
                                    |
                                    v (Biên dịch & Nén mô hình sang INT8)
            +--------------------------------------------------------+
            | GIAI ĐOẠN 2: HỌC CHUYỂN GIAO & CÁ NHÂN HÓA (FINE-TUNE) |
            | - Nguồn dữ liệu: 3 đêm đầu tiên của người dùng Việt    |
            | - Phương pháp: Đóng băng các lớp tích chập của MVP,    |
            |   chỉ tinh chỉnh trọng số lớp Dense trên Gateway       |
            +--------------------------------------------------------+


### Giai đoạn 1: Huấn luyện Mô hình Cơ sở (Pre-training)
Giải pháp: Mô hình học sâu 1D-CNN sẽ được huấn luyện ban đầu trên các cơ sở dữ liệu y tế quốc tế lớn được mở khóa từ PhysioNet như MESA Sleep Database (hơn 2.000 đối tượng đa ký giấc ngủ) và UCD Sleep Apnea Database.

Ý nghĩa: Bước này nhằm chứng minh tính khả thi về mặt kỹ thuật (Technical Feasibility) của hệ thống. Mô hình sẽ học được cấu trúc tổng quát của các biến cố y tế như: mối tương quan sinh lý giữa sự sụt giảm  và sự co mạch ngoại vi thể hiện trên biên độ sóng PPG thô. Các đặc trưng này mang tính phổ quát toàn cầu và không bị ảnh hưởng quá nhiều bởi yếu tố chủng tộc hay biên giới địa lý.

### Giai đoạn 2: Học chuyển giao thích ứng dân số Việt Nam (Transfer Learning & Adaptation)
Giải pháp: Khi sản phẩm được thương mại hóa tại Việt Nam, thiết bị sẽ áp dụng thuật toán thích ứng nhanh. Trong 3 đêm đầu tiên người dùng đeo nhẫn, hệ thống sẽ thực hiện quá trình tự hiệu chuẩn (Self-Calibration).

Kỹ thuật:
1. Trình biên dịch sẽ đóng băng hoàn toàn các trọng số của các lớp tích chập (Conv2D) đóng vai trò trích xuất đặc trưng sinh học trên MVP, do các đặc trưng hình thái sóng mạch đã được học rất tốt từ tập dữ liệu quốc tế.
2. Hệ thống sẽ thu thập các chỉ số BPM và  ở trạng thái tĩnh (khi người dùng ngủ bình thường không có biến cố) để xác định "đường nền sinh học" (Individual Baseline) riêng biệt của người dùng đó (ví dụ: người có thể trạng nhỏ hoặc huyết áp thấp thường có biên độ mạch đập và nhịp tim nền khác biệt).
3. Quá trình tinh chỉnh (Fine-tuning) sẽ chỉ cập nhật trọng số của lớp kết nối đầy đủ (Dense) cuối cùng dựa trên dữ liệu hiệu chuẩn này. Việc tính toán này cực kỳ nhẹ và có thể được thực hiện trực tiếp trên Bộ điều khiển cục bộ (Local Gateway Raspberry Pi 4) thông qua kết nối BLE.

Kết quả: Giải pháp này loại bỏ hoàn toàn yêu cầu phải sở hữu một tập dữ liệu lâm sàng khổng lồ của người Việt ngay từ đầu, đồng thời cải thiện đáng kể sai số tuyệt đối trung bình (MAE) của mô hình, giúp thiết bị thích ứng hoàn hảo với từng cá nhân khách hàng Việt Nam.
