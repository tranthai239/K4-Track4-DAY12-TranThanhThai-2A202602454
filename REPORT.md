# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Trần Thanh Thái |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 2A202602454 |
| GitHub | tranthai239 |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | 2774 / 5061 |
| Số seed huấn luyện bạn dùng (`--seeds`) | 5 |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.274 | 19.1 | BLOCK |
| IDM | 1.533 | 0.1 | PASS (baseline) |
| EgoMLP | 0.751 .. 0.775 (mean 0.763) | 14.1 .. 32.0 (mean 22.1) | BLOCK |
| PercMLP | 0.611 .. 0.620 (mean 0.615) | 0.6 .. 1.5 (mean 1.0) | BLOCK |
| PercMLP+ (sau khi sửa) | 0.815 (mean 0.815) | 0.2 .. 0.2 (mean 0.2) | BLOCK |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Trong Bước 0 kiểm tra dữ liệu (`data_audit.json`), từ 992,679 dòng và 9,993 quỹ đạo thô ban đầu, tôi phát hiện cột gia tốc xe dẫn `Acc_LV` có hệ số tương quan với đạo hàm vận tốc thực tế d(Speed_LV)/dt chỉ đạt xấp xỉ 0.0454 (gần như bằng 0, không có giá trị vật lý), trong khi xe sau `Acc_FAV` có tương quan lên tới 0.9291. Do đó cột `Acc_LV` bị lỗi cảm biến hoặc xử lý sai, hoàn toàn không thể sử dụng. Giải pháp là tái tạo lại gia tốc xe dẫn bằng cách lấy đạo hàm từ chuỗi vận tốc đã làm mượt `Speed_LV` (`derive_lead`), cho ra phân vị giảm tốc p0.1 là -2.77 m/s² và p1 là -1.95 m/s². Ngoài ra, có đúng 383 quỹ đạo bị trùng lặp hoàn toàn (`exact_duplicates_removed = 383`), chỉ còn lại 9,610 quỹ đạo sạch. Ta bắt buộc phải chia tập train/test theo từng quỹ đạo thay vì chia ngẫu nhiên theo khung hình (frame) vì các khung hình trong cùng một chuỗi thời gian 10 Hz có mức tương quan cực kỳ cao; nếu chia theo khung hình sẽ gây rò rỉ dữ liệu (data leakage) nghiêm trọng giữa tập huấn luyện và kiểm thử.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Khi kiểm tra độ tái lập (`replay.json`), bản tái cấu trúc kiểu `float32` khác từng bit ở toàn bộ 600/600 đoạn (100%), nhưng số đoạn vượt dung sai (differ_beyond_tol) là đúng 0 đoạn, với sai lệch tuyệt đối lớn nhất cực nhỏ `max_abs_diff` = 9.32e-05 m/s² (nằm gọn trong ngưỡng `atol = 1e-3`, `rtol = 1e-3`). Điều này xuất phát từ việc phép cộng số thực dấu phẩy động không có tính kết hợp, như minh chứng trong `summation_order_demo` khi cộng 50,000 số float32 xuôi/ngược gây lệch tương đối 6.94e-06. Khi thay đổi logic AEB (`aeb_ttc = 1.5` s), chỉ có đúng 6/600 đoạn bị lệch (các đoạn 73, 78, 179, 297, 319, 341) với sai lệch gia tốc lớn nhất 1.84 m/s². Mở các đoạn này ra cho thấy lệnh phanh chỉ khác biệt ở đúng những khung hình có khoảng cách đang co lại nhanh và TTC tụt xuống dưới 1.5 s, còn các khung an toàn thì giữ nguyên tuyệt đối. Chế độ phát lại nhanh nhất (fastest) che giấu toàn bộ vấn đề trễ hạn khi ghi nhận 0 khung trễ và 0 khung rớt; ngược lại chế độ thời gian thực (realtime) làm lộ ra 582 khung bị trễ (late) và 580 khung bị bỏ rơi (dropped), khiến 371/600 đoạn bị lệch dung sai với sai số cực đại lên đến 5.11 m/s².

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới kịch bản cắt vào được thiết lập ở vận tốc cruise v_ego = 12.0 m/s, cự ly khoảng cách gap từ 2.76 m đến 59.31 m (60 mốc), vận tốc tiến gần dv từ 0.0 đến 12.0 m/s (45 mốc), tạo thành 2,700 ô kịch bản (`scenarios.json`). Khi không có độ trễ phản ứng (t_react = 0.0 s), có 2.22% số ô (tương ứng 60 ô) là vùng va chạm vật lý bất khả thi (unavoidable) mà không một bộ điều khiển nào tránh được dù phanh hết lực 9.0 m/s². Khi đưa độ trễ phản ứng thực tế 0.6 s vào, vùng va chạm bất khả thi và diện tích va chạm của IDM tăng vọt từ 2.22% lên 7.44% (tăng thêm 5.22 điểm phần trăm, gấp hơn 3.3 lần). Phân tích tập log cho thấy dữ liệu lái xe thực tế không hề bao phủ các tình huống khẩn cấp: giới hạn phanh gắt nhất của con người trong log (phân vị 99.9%) chỉ là 3.62 m/s² (trong khi phanh xe chịu được tới 9.0 m/s²), và có tới 8.56% số ô trên lưới kịch bản đòi hỏi mức giảm tốc vượt quá giới hạn phanh cực đại này. Trong toàn bộ 9,610 quỹ đạo chỉ khai thác được 614 sự kiện xe dẫn phanh (chiếm 6.39%), với mức phanh trung vị p50 là -2.53 m/s² và p99 là -4.21 m/s².

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

Bảng so sánh 4 bộ điều khiển từ `closedloop.json` và `openloop.json`:
- PercMLP: L2 vòng hở 0.615 m (Hạng 1) | Va chạm vòng kín 1.0% (Hạng 2)
- EgoMLP: L2 vòng hở 0.763 m (Hạng 2) | Va chạm vòng kín 22.1% (Hạng 4)
- const-vel: L2 vòng hở 1.274 m (Hạng 3) | Va chạm vòng kín 19.1% (Hạng 3)
- IDM: L2 vòng hở 1.533 m (Hạng 4) | Va chạm vòng kín 0.1% (Hạng 1)

Có tới 4 cặp bị đảo ngược thứ hạng hoàn toàn: ('const-vel', 'IDM'), ('const-vel', 'EgoMLP'), ('IDM', 'EgoMLP'), ('IDM', 'PercMLP'). IDM có sai số L2 vòng hở tệ nhất (1.533 m) vì nó là luật điều khiển phản hồi liên tục nhằm giữ cự ly an toàn thay vì bắt chước từng bước vận tốc người lái trong dữ liệu tĩnh; tuy nhiên trong vòng kín, cơ chế phản hồi theo khoảng cách thực giúp IDM phản ứng kịp thời và đạt tỷ lệ va chạm chỉ 0.1% (3/2883 log). Ngược lại, EgoMLP đạt L2 vòng hở rất tốt (0.763 m, vượt trội so với const-vel 1.274 m) vì nó khớp tốt phân bố tốc độ tự thân của người lái trong điều kiện bình thường; nhưng do hoàn toàn không nhìn thấy xe dẫn (thiếu khoảng cách gap và vận tốc xe dẫn), EgoMLP không hề biết xe trước phanh, dẫn đến đâm 22.1% trên toàn bộ log và đâm tới 57.3% trên lát cắt xe dẫn phanh.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Tỷ lệ va chạm vòng kín của mô hình EgoMLP dao động dữ dội qua 5 seed huấn luyện ngẫu nhiên: seed 5065 chỉ đâm 14.1% (407/2883 log), seed 5061 đâm 14.4%, seed 5062 đâm 20.3%, seed 5064 đâm 29.9%, và seed 5063 đâm tới 32.0% (923/2883 log), độ lệch biên độ lên tới 17.9 điểm phần trăm (gấp hơn 2.2 lần). Khoảng tin cậy Wilson 95% khẳng định sự phân tách rõ rệt: seed tốt nhất 5065 có khoảng CI95 là [12.9%, 15.4%], hoàn toàn tách rời và không hề giao thoa với khoảng CI95 của seed xấu nhất 5063 là [30.3%, 33.7%]. Nếu kỹ sư chỉ chạy duy nhất 1 lần ngẫu nhiên và trúng seed 5065, họ sẽ lầm tưởng mô hình hoạt động khá ổn định với tỷ lệ va chạm chỉ 14.1% (thấp hơn nhiều mức 19.1% của constant-velocity) và đưa ra quyết định sai lầm. Việc chỉ báo cáo một seed duy nhất che giấu sự bất ổn định cố hữu do quá trình khởi tạo trọng số ngẫu nhiên trong mạng nơ-ron, tạo ra kết luận sai lệch nghiêm trọng về độ an toàn của hệ thống tự hành.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Thời khoảng cách hiệu chỉnh tốt nhất cho tác nhân xe sau IDM là T = 1.4 s (`agents.json`), đạt khoảng cách phân bố Wasserstein W1 = 0.36 so với phân bố thời khoảng cách người lái thật (trung vị là 1.86 s), thực tế hơn rất nhiều so với xe không phản ứng có W1 = 0.85. 
Bảng so sánh va chạm trên lưới cắt vào (t_react = 0.6 s, cự ly xe sau 18.8 m):
- const-vel: Replay rear 0.0%, Any 77.6% | Reactive rear 0.0%, Any 77.6%
- IDM: Replay rear 68.3%, Any 75.7% | Reactive rear 0.0%, Any 7.4%
- EgoMLP: Replay rear 0.0%, Any 76.9% | Reactive rear 0.0%, Any 76.9%
- PercMLP: Replay rear 44.0%, Any 74.1% | Reactive rear 0.0%, Any 30.1%
- PercMLP+: Replay rear 67.7%, Any 75.1% | Reactive rear 0.0%, Any 11.2%

Khi xe phía sau chỉ phát lại thụ động (replay), nó duy trì tốc độ cũ mà không biết xe thử nghiệm phía trước đang phanh gấp để tránh xe cắt vào. Khi IDM thực hiện phanh khẩn cấp chuẩn xác để tránh va chạm phía trước, xe sau phát lại sẽ đâm thẳng vào đuôi xe IDM, tạo ra 68.3% va chạm phía sau giả mạo và khiến IDM bị tính tổng va chạm lên tới 75.7% (ngang ngửa xe chạy vận tốc cố định 77.6%). Nhưng khi xe sau là tác nhân phản ứng biết quan sát và phanh lại, va chạm phía sau của IDM tụt về đúng 0.0%, bộc lộ tỷ lệ va chạm thật sự chỉ là 7.4% (đúng bằng ngưỡng vật lý bất khả thi). Môi trường thử nghiệm không phản ứng đã trừng phạt bất công hành vi phanh an toàn và làm sai lệch bản chất đánh giá.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

Cổng phát hành ban đầu chặn PercMLP (`gate.json`) bởi 3 quy tắc:
1. R2_closed_loop_logs: Tỷ lệ va chạm vòng kín là 1.5% với cận trên CI95 là 2.0% > baseline IDM 0.1% + 1.0% = 1.1% (nguyên nhân gốc: sai số tích lũy / covariate shift khi tự lái khiến xe trượt vào các trạng thái lạ chưa từng gặp trong log).
2. R4_scenario_grid: Diện tích va chạm tránh được trên lưới cắt vào là 22.6% > baseline 0.0% + 2.0% = 2.0% (nguyên nhân gốc: thiếu dữ liệu kịch bản hiếm do log Argoverse không chứa các ca cắt vào nguy hiểm).
3. R6_reactive_world: Tỷ lệ va chạm trong thế giới phản ứng là 30.1% > baseline 7.4% + 2.0% = 9.4%.

Phương pháp sửa kết hợp 2 kỹ thuật: bổ sung 20,000 kịch bản cắt vào mô phỏng (tạo ra 197,200 cửa sổ) để bù đắp vùng phân bố thiếu, và chạy 2 vòng lặp DAgger trên toàn bộ tập train để thu thập các trạng thái xe tự đi vào rồi gán nhãn bằng dự báo của IDM (tổng cộng 896,569 cửa sổ). Sau khi sửa, PercMLP+ giảm va chạm vòng kín trên log từ 1.5% xuống 0.2% (cận trên 0.4%, vượt qua R2) và giảm diện tích va chạm trên lưới cắt vào từ 22.6% xuống còn 3.7%. Tuy nhiên, PercMLP+ vẫn bị CHẶN tại cổng bởi R4 (3.7% > 2.0%) và R6 (11.2% > 9.4%). PercMLP+ không hề tốt hơn IDM (IDM chỉ đâm 0.1% trên log và 0.0% diện tích tránh được trên lưới); bản chất PercMLP+ chỉ là mô hình học chưng cất lại hành vi của IDM nên không thể vượt qua người thầy của nó. Hiện tại KHÔNG nên phát hành PercMLP+. Đề xuất: Cần bổ sung thêm dữ liệu DAgger tại vùng biên ranh giới va chạm và kiểm tra độ trễ chấp hành thực tế trước khi xem xét lại.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

Ba giới hạn quan trọng nhất của bài lab đối với quyết định phát hành gồm:
1. Giả định chuyển động thuần túy 1 chiều dọc (1-D Longitudinal): Mô phỏng bỏ qua hoàn toàn bình diện ngang (lateral control), chuyển làn, đánh lái né tránh và động lực học quay vòng 2D/3D. Trong thực tế, một tình huống nguy cấp thường được giải quyết bằng kết hợp phanh và đánh lái, hoặc va chạm xảy ra do xe khác tạt đầu chéo góc.
2. Giả định hệ thống nhận thức hoàn hảo (No perception noise / latency / occlusion): Trong bài lab, mô hình nhận trực tiếp giá trị khoảng cách gap và vận tốc xe dẫn chính xác. Trong thực tế, các cảm biến Camera, LiDAR, Radar luôn có độ trễ đo đạc, nhiễu tín hiệu, hiện tượng mất dấu mục tiêu (tracking drop), và điểm mù trong điều kiện mưa gió hay chói sáng. Thử nghiệm bổ sung nhiễu gap ±1.0 m và tốc độ ±0.5 m/s trong bài lab đã cho thấy sự biến động của mô hình.
3. Sự đơn giản hóa hành vi của các phương tiện xung quanh: Xe dẫn trong log chỉ là một chuỗi phát lại cố định không bao giờ phản ứng với xe thử nghiệm, trong khi xe sau chỉ là một luật IDM 1 chiều đơn giản với độ trễ phản ứng cố định 0.6 s. Thực tế hành vi giao thông của con người có sự tương tác qua lại phức tạp và đa dạng về tâm lý lái xe.

Để thu hẹp khoảng cách đến thế giới thực, bước tiếp theo cần: chuyển đổi sang môi trường mô phỏng 3D đa làn đầy đủ vật lý (như CARLA hoặc mô phỏng của hãng), tích hợp toàn bộ pipeline nhận thức từ cảm biến thô (camera image / LiDAR point cloud), thử nghiệm mô hình trong chế độ chạy ngầm (shadow mode) trên đội xe thực tế để so sánh phản xạ với tài xế an toàn trước khi cấp phép điều khiển thực tế.
