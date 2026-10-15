# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Hoàng Đức Dũng |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 2A202602798 |
| GitHub | hddung-vinai |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | 9075 / 9991–9995 |
| Số seed huấn luyện bạn dùng (`--seeds`) | 5 |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.253 | 19.9 (1 lần chạy, luật cố định) | BLOCK (R2, R3, R4, R6) |
| IDM | 1.514 | 0.14 | (baseline) |
| EgoMLP | 0.760 .. 0.764 | 13.8 .. 22.2 | BLOCK (R2, R3, R4, R5, R6) |
| PercMLP | 0.616 .. 0.625 | 0.66 .. 1.35 | BLOCK (R2, R3, R4, R6) |
| PercMLP+ (sau khi sửa) | 0.833 ± 0.025 | 0.35 .. 0.42 | RELEASE |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Vấn đề 1: cột `Acc_LV` không dùng được. Tương quan của nó với đạo hàm `d(Speed_LV)/dt` chỉ là 0.045, trong khi cột gia tốc của xe theo sau tương quan 0.929 với đạo hàm tốc độ của chính nó. Nghĩa là `Acc_LV` gần như không mô tả gia tốc thật của xe dẫn. Vì vậy khi khai thác ca phanh, tôi dùng gia tốc tính từ `Speed_LV` **đã làm mượt** (`logs.a_lead`). Với cách này, gia tốc xe dẫn ở phân vị 1 % là −1.95 m/s², hợp lý về vật lý.
Vấn đề 2: có 383 quỹ đạo trùng hệt nhau nằm ở các id cách xa nhau. Sau khi loại trùng còn 9 610 trên 9 993 quỹ đạo. Nếu không loại, một bản sao có thể rơi vào train và bản còn lại vào test, khiến điểm test bị thổi phồng.
Vấn đề 3: `gap` bằng hiệu vị trí trừ đi một hằng số 4.5 m (chiều dài xe), nên phải hiểu đúng định nghĩa trước khi tính TTC.
Tôi chia theo **quỹ đạo**, vì các khung liên tiếp trong một quỹ đạo (cách nhau 0.1 s) gần như giống nhau. Chia theo khung thì test sẽ chứa "hàng xóm" của train, tức rò rỉ dữ liệu, và đo khả năng nhớ chứ không đo khả năng tổng quát hoá.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Chạy lại cùng mã cho kết quả giống hệt: 0/600 đoạn khác. Bản refactor float32 **khác bit ở cả 600/600 đoạn** nhưng **0 đoạn vượt dung sai** (atol = rtol = 1e-3), sai khác lớn nhất là 4.0e-4 m/s². Đây chỉ là sai số làm tròn: ngay cả `(0.1+0.2)+0.3 ≠ 0.1+(0.2+0.3)`, và tổng float32 của 50k số đổi khoảng 6.9e-6 (tương đối) khi đảo thứ tự cộng. Vì vậy phép so "giống từng bit" là phép thử sai, còn dung sai 1e-3 m/s² thì nhỏ hơn nhiều so với mức có ý nghĩa với lệnh gia tốc.
Thay đổi AEB làm lệch 9/600 đoạn vượt dung sai, tức chỉ một số ít đoạn. Mở đoạn 53 thì chỉ 4/106 khung bị lệch (khung 98–101). Ở đúng các khung đó TTC là 1.43–1.49 s, dưới ngưỡng AEB 1.5 s, và lệnh đổi từ −5.44 thành −6.0 m/s². Đoạn 300 cũng chỉ lệch ở khung 72–73 với TTC 1.43–1.49 s. AEB chỉ kích hoạt khi TTC < 1.5 s, nên các khung khác không đổi.
Phát nhanh nhất có 0 khung trễ và 0 khung rớt trên 59 617 khung. Phát theo thời gian thật có 582 khung trễ hạn và 575 khung rớt, làm 372/600 đoạn lệch vượt dung sai. Phát nhanh nhất luôn chờ bộ điều khiển tính xong nên che mất lỗi trễ hạn tính toán. Ngoài đời, thế giới không chờ, và lệnh cũ bị giữ lại.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới cắt vào của tôi có tốc độ cruise 14.0 m/s, khoảng cách 2.76–59.3 m, tốc độ tiến gần 0–14 m/s, tổng cộng 2 700 ô. Với độ trễ 0 s, 3.6 % số ô là **bất khả thi** (gap < dv²/(2·b_max)). Với độ trễ 0.6 s, con số này tăng lên 10.2 %. Bộ mô phỏng phanh hết cỡ khớp hoàn toàn với biên `unavoidable` của tôi: 0 ô lệch ở cả hai độ trễ.
Vùng đâm của IDM tăng từ 3.6 % lên 10.2 %, đúng bằng vùng bất khả thi, nên diện tích va chạm "do bộ điều khiển" của IDM là 0 % ở cả hai độ trễ. Ngược lại, const-vel đâm 80.6 % số ô, trong đó 70.4 % là tránh được (độ trễ 0.6 s). Biên vật lý giúp tách "ai cũng đâm" khỏi "bộ điều khiển thất bại".
Log **không phủ** vùng khó. Tôi khai thác được 614 ca xe dẫn phanh (6.4 % số quỹ đạo). Phanh mạnh nhất ở trung vị chỉ 2.53 m/s², ở phân vị 99 là 4.21 m/s², và giới hạn phanh của người lái (p99.9) là 3.62 m/s². Trong khi đó 12.7 % số ô của lưới đòi giảm tốc mạnh hơn mức này. Một mô hình chỉ học từ log chưa từng thấy vùng đó.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

| Bộ điều khiển | L2 (m) | Hạng L2 | Va chạm (trung bình seed) | Hạng va chạm |
|---|---|---|---|---|
| PercMLP | 0.620 | 1 | 1.03 % | 2 |
| EgoMLP | 0.762 | 2 | 16.9 % | 3 |
| const-vel | 1.253 | 3 | 19.9 % | 4 |
| IDM | 1.514 | 4 | 0.14 % | 1 |

Có ba cặp đổi thứ tự: (const-vel, IDM), (IDM, EgoMLP) và (IDM, PercMLP). IDM xếp **cuối** theo L2 nhưng **đầu** theo va chạm.
IDM có L2 lớn vì L2 đo độ giống **phong cách** của người lái trong log. IDM là luật có thời khoảng cách riêng và dự báo giả định xe dẫn giữ tốc độ, nên đường đi của nó lệch khỏi người lái. Nhưng ở vòng kín, IDM phản ứng với khoảng cách thực tế: trung vị lực phanh mạnh nhất là 0.96 m/s², và nó đâm 4/2 883 quỹ đạo.
EgoMLP chỉ nhìn tốc độ của chính nó, **không thấy khoảng cách và tốc độ xe dẫn**. Phần lớn thời gian người lái chạy đều nên ngoại suy tốc độ cho L2 tốt (0.762, so với 1.253 của const-vel). Nhưng trên lát cắt phanh, L2 tăng lên 1.04. Ở vòng kín, khi xe dẫn phanh, EgoMLP không có tín hiệu để phản ứng (trung vị lực phanh mạnh nhất chỉ 0.08 m/s²), nên đâm 52 % trên lát cắt phanh. Lát cắt này chỉ chiếm 5.1 % số cửa sổ, nên bị trung bình L2 che mất.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Tỷ lệ va chạm của EgoMLP qua 5 seed là 14.4 / 22.2 / 13.8 / 17.3 / 16.9 %, tức dải 13.8–22.2 %. Khoảng Wilson 95 % của seed tốt nhất là [12.6, 15.1] %, của seed xấu nhất là [20.8, 23.8] %, **không chồng nhau**. Như vậy sự khác biệt giữa các seed là thật chứ không do nhiễu lấy mẫu: khoảng tin cậy chỉ đo độ bất định của một mô hình, không đo độ bất định do huấn luyện.
Nguy hiểm hơn là PercMLP. Seed 9994 có tỷ lệ va chạm 0.66 % với cận trên 1.03 %, **qua** ngưỡng R2 (1.14 %). Seed 9991 có 1.35 % với cận trên 1.84 %, **rớt**. Nếu chỉ chạy một seed may mắn, tôi đã kết luận PercMLP không kém hơn IDM trên log. Vì vậy cổng chấm theo **seed xấu nhất** và có R5 kiểm tra độ ổn định qua seed.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Bảng dưới đây dùng độ trễ phản ứng 0.6 s; con số là tỷ lệ ô lưới có va chạm bất kỳ (phía trước hoặc phía sau).

| Bộ điều khiển | Xe sau phát lại | Xe sau phản ứng | Va chạm phía sau (phát lại / phản ứng) |
|---|---|---|---|
| const-vel | 80.6 % | 80.6 % | 0.0 / 0.0 % |
| IDM | 80.9 % | 10.2 % | 70.6 / 0.0 % |
| EgoMLP | 80.9 % | 80.9 % | 0.0 / 0.0 % |
| PercMLP | 78.1 % | 34.3 % | 44.1 / 0.0 % |
| PercMLP+ | 80.6 % | 10.6 % | 70.0 / 0.0 % |

Người lái trong log giữ thời khoảng cách trung vị 1.85 s. Giá trị `T = 1.4 s` cho phân bố thời khoảng cách gần người thật nhất (W1 = 0.35), so với 0.88 của một xe sau không bao giờ phản ứng, tức thực tế hơn khoảng 2.5 lần.
Với xe sau phát lại, IDM phanh đúng lúc trước xe cắt vào, nhưng xe sau vẫn chạy theo log như thể không có gì xảy ra và đâm vào đuôi. Đó là **va chạm giả**, chiếm 70.6 % số ô. Trong thế giới này mọi bộ điều khiển đều quanh 80 %, và PercMLP còn trông "tốt nhất" (78.1 %) chỉ vì nó phanh ít hơn. Khi xe sau phản ứng, IDM chỉ còn 10.2 %, đúng bằng vùng bất khả thi. Kết luận đảo ngược: phát lại tác nhân phạt chính bộ điều khiển phanh đúng.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

PercMLP bị chặn bởi bốn quy tắc, xuất phát từ hai nguyên nhân gốc:
- **Kịch bản log không chứa** (R4, R6). Diện tích va chạm tránh được trên lưới cắt vào là 24.1 %, so với 0 % của IDM. Trong thế giới có xe sau phản ứng, tỷ lệ va chạm là 34.3 %, so với 10.2 %. Log chỉ có phanh tới khoảng 3.6 m/s², nên mô hình chưa từng thấy cảnh cắt vào gần.
- **Sai số tích luỹ ở vòng kín** (R2, R3). Trên log thật, cận trên tỷ lệ va chạm là 1.84 %, vượt ngưỡng 1.14 %. Trên lát cắt phanh là 5.1 % (cận trên 9.0 %), dù L2 vòng hở tốt nhất (0.620). Mô hình tự lái lệch dần vào những trạng thái người lái không bao giờ đi vào.

Cách sửa: thêm 197 030 cửa sổ từ 20 000 cảnh cắt vào mô phỏng (nhãn do IDM) để trị nguyên nhân thứ nhất, và chạy 2 vòng DAgger, gắn nhãn IDM ở **chính các trạng thái mô hình tự đi vào**, để trị nguyên nhân thứ hai. Tổng cộng khoảng 894k cửa sổ.
Kết quả: PercMLP+ qua cả 7 quy tắc. Tỷ lệ va chạm trên log là 0.35–0.42 % (cận trên 0.73 %), trên lát cắt phanh là 1.0 %, trên lưới cắt vào là 0.33 %, và trong thế giới phản ứng là 10.6 %. Ở tốc độ 20 m/s chưa từng huấn luyện, nó đạt 0.4–1.8 %, so với 22.6–43.0 % của PercMLP. L2 vòng hở lại **tệ đi** từ 0.620 lên 0.833, thêm một bằng chứng rằng L2 không đo an toàn.
PercMLP+ **không tốt hơn** IDM. Nó chỉ "không kém hơn trong biên cho phép": 0.40 % so với 0.14 % trên log, 0.33 % so với 0 % trên lưới, 10.6 % so với 10.2 % trong thế giới phản ứng. Nó học nhãn từ chính IDM, nên khó vượt thầy. Tôi **không** đề xuất thay IDM ngay: chạy PercMLP+ ở chế độ bóng (shadow mode) song song với IDM, mở rộng họ kịch bản (2D, chuyển làn, cảm biến nhiễu), và chỉ cân nhắc phát hành khi có lợi ích rõ ràng mà IDM không có.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

1. **Thế giới mô phỏng là giả định của chúng ta.** Vật lý một chiều, b_max cố định, IDM vừa là baseline, vừa là "thầy" dán nhãn, vừa là mô hình xe sau. PercMLP+ qua cổng một phần vì nó được chấm trong cùng thế giới đã sinh ra dữ liệu sửa lỗi của nó, nên vòng lập luận này có thể tự khẳng định chính nó.
2. **Không có cảm biến thật.** Mô hình nhận gap và tốc độ xe dẫn gần như đúng tuyệt đối, chỉ có nhiễu Gauss nhỏ (1.0 m và 0.5 m/s). Mất dấu, nhận nhầm, trễ cảm biến hay các lỗi tương quan theo thời gian đều không được kiểm thử, trong khi bước 1 cho thấy chỉ riêng trễ tính toán đã làm 372/600 đoạn lệch.
3. **Xe dẫn trong log không phản ứng, và chỉ có bám xe một chiều.** Không có chuyển làn, người đi bộ hay giao lộ. Log Argoverse 2 chỉ dài 7–11 s và phanh không quá khoảng 3.6 m/s². Tỷ lệ va chạm 0.4 % trên 2 883 quỹ đạo không nói gì về sự kiện hiếm cỡ 1e-6 trên mỗi km.
4. **Số liệu thống kê còn mỏng.** Lát cắt phanh chỉ có 198 quỹ đạo nên khoảng tin cậy rộng (cận trên của PercMLP+ là 3.6 %), và 5 seed chưa đủ để ước lượng đuôi phân bố huấn luyện.

Việc cần làm tiếp: kiểm thử với dữ liệu và bộ mô phỏng độc lập (tác nhân học từ dữ liệu, không phải IDM), thêm mô hình lỗi cảm biến và trễ thời gian thực, mở rộng sang kịch bản 2D, chạy shadow mode trên xe thật để thu dữ liệu ngoài phân bố, và tăng số seed và số kịch bản để thu hẹp khoảng tin cậy.
