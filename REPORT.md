# REPORT — Lab Day 12

| Thông tin | Giá trị |
|---|---|
| Họ và tên | Bùi Đức Thông |
| MSSV | 2A202602931 |
| GitHub | thongbuind |
| Seed chia dữ liệu / seed mô hình | 8128 / 4957; các seed 4957, 4958, 4959, 4960, 4961 |
| Số seed huấn luyện (`--seeds`) | 5 |

Chạy đầy đủ bằng `python run_all.py --student-id 2A202602931`, không dùng `--quick`.
Mọi kết quả dưới đây lấy từ pipeline của MSSV này; dải biểu diễn min–max qua 5 seed, không phải khoảng tin cậy.
Đối với PercMLP+, dải L2 được tính lại bằng `l2_summary` trên đúng tập kiểm thử và 5 checkpoint do bước fix sinh ra, vì `openloop.json` chỉ lưu trung bình và độ lệch chuẩn của mô hình này.

## Kết quả cuối

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.2529 | 20.708 | BLOCK |
| IDM | 1.5447 | 0.104 | baseline |
| EgoMLP | 0.7599–0.7727 | 18.626–29.691 | BLOCK |
| PercMLP | 0.6218–0.6314 | 0.728–5.099 | BLOCK |
| PercMLP+ (sau khi sửa) | 0.8060–0.8638 | 0.243–0.277 | RELEASE |

Nguồn: [openloop.json](evidence/openloop.json), [closedloop.json](evidence/closedloop.json), [gate.json](evidence/gate.json).
Cổng sử dụng seed xấu nhất và cận trên Wilson, nên quyết định không chỉ dựa vào trung bình.

## Câu 1 · Dữ liệu

Kiểm toán [data_audit.json](evidence/data_audit.json) ghi nhận 992.679 hàng, 9.993 quỹ đạo thô và loại 383 bản sao chính xác, còn 9.610 quỹ đạo sạch.
Các quỹ đạo dài 70–109 bước (trung vị 106) ở 10 Hz; độ dài không đồng nhất cần được che bằng mask để không tính phần đệm NaN.
Tương quan của `Acc_LV` với đạo hàm vận tốc chỉ 0,04535, trong khi phía follower đạt 0,92909, nên không dùng cột gia tốc lead để khai phá phanh.
Thay thế bằng `a_lead` suy ra từ `Speed_LV` đã làm mượt; phân vị 0,1% gia tốc lead là −2,7691 m/s², tránh coi nhiễu vận tốc là phanh thật.
Khoảng cách thỏa `Pos_LV - Pos_FAV - 4,5` với sai lệch lớn nhất khoảng 3,64e−11 m; đây là kiểm tra nhất quán hình học, không phải bằng chứng vị trí tuyệt đối chính xác.
Chia theo quỹ đạo sau khử trùng lặp để các cửa sổ chồng lấn của cùng một đoạn không lọt vào cả train và test; seed 8128 cho 2.883 quỹ đạo kiểm thử và 71.667 cửa sổ test.

## Câu 2 · Phát lại

Trong [replay.json](evidence/replay.json), chạy lại cùng mã cho 0/600 đoạn khác; bản float32 đổi thứ tự phép toán khác giá trị chính xác ở 600/600 đoạn nhưng 0 đoạn vượt dung sai.
Sai khác tuyệt đối lớn nhất của refactor là 0,000403132 m/s²; so theo `atol + rtol * abs(golden)` với cả hai dung sai 0,001 giúp tránh nhầm làm tròn với thay đổi hành vi.
Bổ sung AEB làm 11/600 đoạn vượt dung sai, sai khác lớn nhất 1,733781 m/s², vì chỉ kích hoạt khi đang tiến gần và TTC dưới 1,5 s, đồng thời lệnh cũ chưa phanh tới −6 m/s².
Tôi kiểm tra đoạn 76 trong log sinh ra: tại 7,32 s, ego 3,1051 m/s, lead 0,3878 m/s, gap 3,7574 m, TTC 1,38277 s; lệnh plain −5,580414 chuyển thành −6,0 m/s².
Phát nhanh nhất báo 0 lệnh trễ và 0 khung rơi; mô phỏng thời gian thật trên cùng 59.516 khung có 581 lệnh trễ, 576 khung rơi và 372/600 đoạn vượt dung sai.
Vì lệnh trước phải được giữ khi xử lý không kịp, tính đúng của công thức chưa đủ: replay cần kiểm tra cả deadline, thứ tự và độ trễ thông điệp; chi phí ở đây được mô phỏng, không phải đo latency phần cứng thật.

## Câu 3 · Kịch bản

Theo [scenarios.json](evidence/scenarios.json), lưới dùng tốc độ cruise 12 m/s, gap 2,76005–59,31233 m và closing speed 0–12 m/s, gồm 60 × 45 = 2.700 ô.
Biên vật lý là `gap < dv*t_react + dv²/(2*b_max)`, với `b_max = 9 m/s²`: trước khi phản ứng mất `dv*t_react`, sau đó cần quãng phanh để triệt tiêu vận tốc tương đối.
Tỷ lệ bất khả kháng tăng từ 2,222% tại trễ 0 s lên 7,444% tại 0,6 s; IDM tăng đúng từ 2,222% lên 7,444%, tương đương 5,222 điểm phần trăm, khoảng 3,35 lần.
Full-brake và biên giải tích có 0 ô sai khác ngoài vùng sát biên ở cả hai độ trễ (lần lượt 2.690 và 2.673 ô được so), giúp kiểm chứng mô phỏng trước khi quy lỗi cho planner.
Khai phá được 614 sự kiện phanh lead (6,389% quỹ đạo); giới hạn phanh follower ở phân vị 99,9% là 3,62484 m/s², thấp đáng kể so với khả năng cơ cấu chấp hành 9 m/s².
Có 8,556% lưới đòi phanh mạnh hơn ngưỡng người lái này, nên dữ liệu log không bao phủ đủ độ khó của stress test; con số phân vị 99,9% cũng không phải giá trị cực đại tuyệt đối của mọi mẫu.

## Câu 4 · Vòng hở và vòng kín

| Bộ điều khiển | L2 trung bình (m) | Hạng L2 | Va chạm trung bình qua seed (%) | Hạng vòng kín |
|---|---|---|---|---|
| const-vel | 1.2529 | 3 | 20.708 | 3 |
| IDM | 1.5447 | 4 | 0.104 | 1 |
| EgoMLP | 0.7660 | 2 | 24.315 | 4 |
| PercMLP | 0.6264 | 1 | 1.824 | 2 |

Nguồn bảng và thứ hạng: [closedloop.json](evidence/closedloop.json), mục `ranking`; hạng dùng trung bình qua seed, còn cổng dùng seed xấu nhất.
Có 4 cặp đảo thứ tự: constant-velocity/IDM, constant-velocity/EgoMLP, IDM/EgoMLP và IDM/PercMLP; điển hình IDM từ hạng 4 vòng hở lên hạng 1 vòng kín.
IDM có L2 1,5447 m vì giữ khoảng cách theo luật an toàn và giả định lead giữ vận tốc, không tối ưu bắt chước chính xác quỹ đạo người; nó chỉ va chạm 3/2.883 đoạn (0,104%).
EgoMLP có L2 trung bình 0,7660 m tốt hơn constant-velocity 1,2529 m, nhưng không thấy gap hoặc tốc độ lead; khi tự điều khiển, sai lệch làm trạng thái rời phân phối đã quan sát và tích lũy thành va chạm trung bình 24,315%.
Lát cắt lead sắp phanh chỉ chiếm 4,645% cửa sổ; EgoMLP có L2 lát cắt 1,0747 m so với 0,7660 m toàn bộ, nên một trung bình đẹp dễ che tình huống quan trọng.

## Câu 5 · Một lần chạy chưa đủ

Trong [closedloop.json](evidence/closedloop.json), EgoMLP qua seed 4957–4961 có tỷ lệ va chạm lần lượt 24,592%, 27,159%, 21,505%, 29,691% và 18,626% trên cùng 2.883 quỹ đạo test.
Biên độ là 11,065 điểm phần trăm, vượt xa giới hạn ổn định 1 điểm của R5, dù L2 chỉ dao động 0,7599–0,7727 m.
Seed tốt nhất 4961 có 537/2.883 va chạm và Wilson 95% khoảng [17,247%; 20,089%]; seed xấu nhất 4960 có 856/2.883 và khoảng [28,051%; 31,385%].
Wilson mô tả bất định lấy mẫu tỷ lệ va chạm của một mô hình cố định, không thay thế đo độ biến động do khởi tạo và huấn luyện; cũng cần thận trọng vì các quỹ đạo giao thông có thể chưa độc lập hoàn toàn.
Chỉ chọn seed 4961 có thể báo 18,626%, còn chọn 4960 lại báo 29,691%; chênh 11,065 điểm làm kết luận phụ thuộc may rủi hoặc chọn lọc kết quả.
Do đó báo cả 5 seed, khoảng tin cậy và seed xấu nhất; R2 dùng cận trên 31,385% của EgoMLP so với ngưỡng IDM 0,104% + 1 điểm, nên bị chặn rõ ràng.

## Câu 6 · Tác nhân phản ứng

| Bộ điều khiển | Xe sau phát lại: mọi va chạm (%) | Xe sau phản ứng: mọi va chạm (%) | Va chạm sau: phát lại / phản ứng (%) |
|---|---|---|---|
| const-vel | 77.630 | 77.630 | 0.000 / 0.000 |
| IDM | 75.741 | 7.444 | 68.296 / 0.000 |
| EgoMLP | 79.074 | 79.074 | 0.000 / 0.000 |
| PercMLP | 77.148 | 33.852 | 45.333 / 0.000 |
| PercMLP+ | 75.185 | 7.630 | 67.556 / 0.000 |

Nguồn: [agents.json](evidence/agents.json), `rear_comparison_t_react_0.6`; các tỷ lệ tính trên toàn bộ lưới và lấy tình huống xấu nhất qua seed cho mô hình học.
Hiệu chỉnh chọn `T = 1,4 s` với Wasserstein headway 0,34454 s so với 0,83954 s của tác nhân không phản ứng; headway trung vị của người là 1,86635 s, nên tác nhân phản ứng khớp log tốt hơn theo phép đo này.
Xe sau cách 18,8 m và trễ 0,6 s: IDM bị tính 68,296% va chạm sau khi xe sau cứ giữ vận tốc, nhưng 0% khi xe sau phanh theo ego; tổng va chạm giảm từ 75,741% xuống 7,444%.
Đây là va chạm giả do giả định hành vi của thế giới: planner phanh an toàn phía trước lại bị xe sau phát lại đâm, làm đánh giá đánh đồng lỗi mô hình môi trường với lỗi điều khiển.
Khi cho xe sau phản ứng, IDM tốt nhất ở 7,444% so với PercMLP 33,852%; tuy vậy khớp một phân phối headway không xác nhận mọi phản ứng của người lái, và va chạm trước/sau là sự kiện cạnh tranh nên không trừ hai tỷ lệ để suy ra một quan hệ nhân quả độc lập.

## Câu 7 · Cổng phát hành và sửa

[gate.json](evidence/gate.json) chặn PercMLP bởi R2_closed_loop_logs, R3_braking_slice, R4_scenario_grid, R5_seed_stability, R6_reactive_world; riêng R2 có cận trên 5,963% > 1,104%, R3 có cận trên 28,649% > 5,508%, và R4 có diện tích tránh được nhưng vẫn đâm 26,407% > 2%.
Căn nguyên gồm thiếu tình huống cut-in và covariate shift: 8,556% lưới đòi phanh mạnh hơn mức phân vị 99,9% trong log, còn PercMLP dao động 0,728–5,099% va chạm giữa seed và đạt 33,852% va chạm trong thế giới phản ứng.
[fix.json](evidence/fix.json) thêm 20.000 cut-in (197.250 cửa sổ tổng hợp) và 2 vòng DAgger; `dagger_windows` cho chính learner lái train logs rồi gắn nhãn dự báo IDM ở đúng trạng thái nó đã đi tới, để dạy phục hồi sau sai số của bản thân.
PercMLP+ đạt 7/7 quy tắc: tỷ lệ va chạm log 0.243–0.277%, cận trên Wilson seed xấu nhất 0.547%, và R4 còn 0.185%; L2 trung bình thành 0.8299 m, nên điểm bắt chước có thể xấu đi khi hành vi an toàn hơn.
Qua cổng chỉ chứng minh không kém baseline trong các bài kiểm tra với biên đã chọn: IDM vẫn đạt 0,104% va chạm log và 0% diện tích lỗi tránh được ở lưới chuẩn; học từ IDM cũng không tự tạo chứng cứ vượt thầy.
Ở cruise chưa thấy 20 m/s (train tổng hợp tối đa 16 m/s), PercMLP+ vẫn đâm trong 0.148–1.556% lưới vốn tránh được; tôi chỉ chấp nhận RELEASE theo cổng của lab, chưa khuyến nghị triển khai thật trước kiểm thử ngoài phân phối, nhiều môi trường và đánh giá shadow có giám sát.

## Câu 8 · Bài lab này không chứng minh được gì

Thứ nhất, mô phỏng chỉ 1 chiều, `dt = 0,1 s` và giới hạn phanh 9 m/s²: không có đánh lái, độ bám thay đổi hay trễ cơ cấu phức tạp; cần mô hình xe đã hiệu chuẩn và thử thêm đường cong, mặt đường xấu, mưa và sự cố chấp hành.
Thứ hai, log 10 Hz được chuyển thẳng thành gap/vận tốc: thử nhiễu gap 1 m và lead speed 0,5 m/s chưa mô tả phát hiện sai, mất vật thể, che khuất hoặc lỗi cảm biến tương quan; cần kiểm thử chuỗi perception và latency thực tế từ đầu đến cuối.
Thứ ba, lead trong log không phản ứng và rear chỉ là IDM với T 1,4 s, trễ 0,6 s; việc va chạm sau về 0% không chứng minh người thật sẽ phản ứng như vậy, nên cần nhiều mô hình người lái và mô phỏng tương tác hai chiều được đối chiếu dữ liệu.
Thứ tư, tập test chỉ 2.883 quỹ đạo và 197 đoạn lead phanh, lại cùng nguồn Argoverse: 5 seed kiểm tra bất định huấn luyện trên một split 8128, chưa bao trùm bất định địa điểm, ngày, lưu lượng hay các mẫu phụ thuộc; cần chia theo bối cảnh và dữ liệu ngoài miền.
Thứ năm, 2.700 ô không đại diện đầy đủ thế giới liên tục; kết quả tốc độ 20 m/s của PercMLP+ còn 0.148–1.556% diện tích lỗi tránh được cho thấy vượt cổng trên miền 12 m/s không bảo đảm tổng quát hóa.
Cuối cùng, các ngưỡng +1, +5 và +2 điểm phần trăm là tiêu chuẩn bài lab, không phải chứng nhận an toàn đường bộ; cần kiểm chứng độ phủ, tìm ca đối kháng, đánh giá comfort và kế hoạch fallback rồi mới cân nhắc thử nghiệm thực tế có giám sát.
