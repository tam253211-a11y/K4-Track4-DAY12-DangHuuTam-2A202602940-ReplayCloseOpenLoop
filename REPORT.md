# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Đặng Hữu Tâm |
| MSSV | 2A202602940 |
| GitHub | tam253211-a11y |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | split seed 3464 / model seed 7634 (5 seed huấn luyện: 7634–7638) |
| Số seed huấn luyện bạn dùng (`--seeds`) | 5 (mặc định) |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.286 | 21.4 | BLOCK (R2, R3, R4, R6) |
| IDM | 1.566 | 0.1 | (baseline) |
| EgoMLP | 0.761 .. 0.771 (TB 0.767) | 18.4 .. 33.0 | BLOCK (R2–R6) |
| PercMLP | 0.621 .. 0.635 (TB 0.626) | 0.6 .. 6.5 | BLOCK (R2–R6) |
| PercMLP+ (sau khi sửa) | 0.830 ± 0.016 (TB qua 5 seed) | 0.3 .. 0.3 (8–9 / 2883) | RELEASE (qua cả 7 quy tắc) |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Ba vấn đề. (1) Cột `Acc_LV` gần như không liên quan đến đạo hàm của `Speed_LV`: tương quan chỉ **0.045**, trong khi cột tương ứng của xe theo sau đạt **0.929**. Nếu dùng `Acc_LV` để khai thác ca phanh, ta sẽ bắt nhầm nhiễu, nên tôi dùng gia tốc xe dẫn **tự suy ra từ `Speed_LV` đã làm mượt** (`logs.a_lead`, p0.1 = −2.77 m/s²). (2) Có **383** quỹ đạo trùng hệt nhau nằm ở các ID cách xa nhau; sau khi loại còn **9610/9993** quỹ đạo. Nếu giữ lại, một bản sẽ rơi vào train và bản kia vào test: mô hình được chấm trên chính dữ liệu đã thấy nên điểm bị thổi phồng. (3) Rò rỉ theo thời gian: các khung liên tiếp ở 10 Hz gần như giống nhau, nên chia theo khung thì cửa sổ test chỉ là "hàng xóm" của cửa sổ train. Vì vậy phải chia theo **quỹ đạo**, để toàn bộ một quỹ đạo nằm trọn một phía. (Kiểm tra thêm: `Spatial_Gap` = hiệu vị trí − 4.5 m chiều dài xe, sai lệch tối đa ~4e-11 m, nên cột khoảng cách tin được.)

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Log có 600 đoạn (9.6 MB). Chạy lại cùng mã: 0/600 đoạn khác. Bản refactor float32 đổi thứ tự phép tính: **600/600 đoạn không giống từng bit** nhưng **0/600 vượt dung sai** (atol = rtol = 1e-3, lệch lớn nhất 9.7e-05 m/s²). Phép cộng số thực không kết hợp, (0.1+0.2)+0.3 = 0.6000000000000001 ≠ 0.6, nên so `==` sẽ báo lỗi giả ở mọi đoạn; dung sai 1e-3 m/s² nhỏ hơn rất nhiều so với mức ảnh hưởng tới chuyển động của xe. Thay đổi AEB làm lệch **8/600** đoạn (126, 232, 237, 260, 425, 455, 514, 541). Mở đoạn 126: chỉ các khung **82–85** (trên 86) bị lệch, với TTC lần lượt 1.49, 1.44, 1.43, 1.36 s, tức đều **< 1.5 s**, đúng ngưỡng kích hoạt AEB; lệnh đổi từ −4.44..−5.15 thành −6.0 m/s². Ở các khung khác TTC lớn hơn nên quy tắc AEB không bao giờ kích hoạt, và lệnh giống hệt bản cũ. Phát nhanh nhất cho **0 khung trễ, 0 khung rớt** vì không có khái niệm hạn chót. Phát theo thời gian thật lộ ra **586 khung trễ + 582 khung rớt** trên 60095 khung, làm **382/600** đoạn đổi lệnh: một bộ điều khiển đúng về toán vẫn có thể trễ hạn trên xe thật.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới cắt vào với tốc độ cruise **10.0 m/s**, khoảng cách 2.76–59.31 m (p2–p98 của log), tốc độ tiến gần 0–10 m/s: 60 × 45 = **2700 ô**. Vùng bất khả thi (phanh 9 m/s² sau trễ vẫn đâm) chiếm **1.0 %** lưới khi t_react = 0 và **5.1 %** khi t_react = 0.6 s. Kiểm chứng bằng bộ phanh hết cỡ: 0 ô lệch ngoài biên ở cả hai độ trễ. Vùng đâm của IDM tăng từ **1.0 % lên 5.1 %**, và trùng **khít** vùng bất khả thi (diện tích "tránh được mà vẫn đâm" = 0.0 %). Như vậy mọi cú đâm của IDM đều là "ai cũng đâm", không phải lỗi của IDM. Ngược lại, const-vel có 68.2 % lưới "tránh được mà vẫn đâm" (t_react 0.6). Log **không phủ vùng khó**: người lái trong log phanh mạnh nhất cỡ **3.6 m/s²** (p99.9), trong khi **5.2 %** lưới đòi phanh mạnh hơn mức đó. 614 ca phanh khai thác được có đỉnh giảm tốc của xe dẫn p50 = −2.5, p99 = −4.2 m/s², nên các chấm khai thác dồn về vùng dễ; mô hình học từ log chưa từng thấy vùng khó này.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

| Bộ điều khiển | L2 vòng hở (m) | Hạng | Va chạm vòng kín (TB qua seed) | Hạng |
|---|---|---|---|---|
| const-vel | 1.286 | 3 | 21.4 % | 3 |
| IDM | 1.566 | 4 | 0.1 % | 1 |
| EgoMLP | 0.767 | 2 | 24.2 % | 4 |
| PercMLP | 0.626 | 1 | 2.7 % | 2 |

Bốn cặp đổi thứ tự: (const-vel, IDM), (const-vel, EgoMLP), (IDM, EgoMLP), (IDM, PercMLP). **IDM** có L2 tệ nhất (1.566 m) vì L2 đo việc *bắt chước người lái*: IDM không lái giống người (giữ khoảng cách theo luật riêng), nhưng luật đó **phản ứng với khoảng cách và tốc độ tiến gần**, nên khi tự lái nó giữ an toàn (0.1 %, khoảng cách tối thiểu trung vị 9.32 m so với người lái 9.40 m). Ngay cả trên lát cắt xe dẫn phanh, L2 của IDM (1.538) vẫn thấp hơn const-vel (1.942). **EgoMLP** có L2 tốt hơn const-vel (0.767 so với 1.286) vì nó học được quán tính tốc độ của người lái, nhưng nó **không thấy khoảng cách và tốc độ xe dẫn**. Trên log, điều đó không sao vì trạng thái luôn là của người lái thật. Khi tự lái, nó không biết xe trước đang phanh, nên không phanh theo, và sai số tích luỹ thành va chạm: 61.1 % trên lát cắt xe dẫn phanh (seed xấu nhất). Vòng hở chấm "dự báo giống người"; vòng kín chấm "hệ quả của chính hành động mình". Hai thứ đó khác nhau.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Qua 5 seed huấn luyện, EgoMLP va chạm từ **18.4 %** (seed 7636, CI95 [17.0, 19.8] %) đến **33.0 %** (seed 7637, CI95 [31.3, 34.8] %), trong khi L2 vòng hở gần như không đổi (0.761..0.771 m). Hai khoảng tin cậy **không chồng nhau**. Wilson (n = 2883) chỉ đo sai số *lấy mẫu trên tập log*, ±~1.5 điểm, nhưng biến thiên do *seed huấn luyện* lớn gấp nhiều lần (14.6 điểm). Vì vậy CI của một seed tạo cảm giác chắc chắn giả. PercMLP cũng vậy: 0.6 % (seed 7635, CI [0.4, 1.0] %) đến 6.5 % (seed 7636, CI [5.6, 7.4] %). Nếu chạy đúng seed 7635, ta có thể báo "PercMLP chỉ 0.6 %, cận trên 1.0 %, gần bằng IDM + 1 điểm", đủ để lập luận cho phát hành, trong khi một seed khác cùng kiến trúc đâm gấp ~10 lần. Do đó cổng dùng **seed xấu nhất** và quy tắc R5 đòi dải qua seed ≤ 1 điểm.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Người lái trong log giữ thời khoảng cách trung vị 1.85 s. Hiệu chỉnh IDM xe sau theo khoảng cách Wasserstein-1 giữa phân bố thời khoảng cách: **T = 1.4 s** (W1 = 0.34), so với **0.86** của xe không bao giờ phản ứng, tức gần với người thật hơn ~2.5 lần.

Lưới cắt vào, t_react = 0.6 s, % ô có va chạm bất kỳ (trước hoặc sau):

| Bộ điều khiển | Xe sau phát lại | Xe sau phản ứng | Va chạm sau: phát lại / phản ứng |
|---|---|---|---|
| const-vel | 73.4 % | 73.4 % | 0.0 % / 0.0 % |
| IDM | 70.1 % | 5.1 % | 64.9 % / 0.0 % |
| EgoMLP | 72.9 % | 72.9 % | 0.0 % / 0.0 % |
| PercMLP | 71.5 % | 32.9 % | 38.7 % / 0.0 % |

Xe sau phát lại chạy theo tốc độ trong log và không biết xe thử vừa phanh, nên **chính hành vi đúng** (IDM phanh khi bị cắt vào) lại bị tính là va chạm phía sau: **64.9 %** lưới, hoàn toàn giả. Trong thế giới phát lại, IDM (70.1 %) trông **ngang ngửa** const-vel (73.4 %), một bộ không bao giờ phanh và vì thế không bao giờ bị đâm sau. Khi xe sau phản ứng, va chạm sau về **0.0 %** và IDM chỉ còn 5.1 %, đúng bằng vùng bất khả thi. Như vậy thế giới không phản ứng **phạt đúng bộ điều khiển an toàn** và đảo ngược kết luận.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

PercMLP bị **BLOCK** vì rớt 5 quy tắc: R2 (seed xấu nhất 6.5 %, cận trên 7.4 % so với IDM 0.1 % + 1 điểm), R3 (21.2 %, cận trên 27.5 % trên lát cắt xe dẫn phanh), R4 (27.7 % lưới "tránh được mà vẫn đâm" so với 0.0 % + 2 điểm), R5 (0.6..6.5 % qua 5 seed), R6 (32.9 % so với 5.1 %). Có hai nguyên nhân gốc. (a) **Kịch bản log không chứa** (R4, R6): log không có cắt vào ở cự ly gần, người lái chỉ phanh tới 3.6 m/s² trong khi 5.2 % lưới đòi hơn thế, nên mô hình chưa từng thấy vùng này. (b) **Sai số tích luỹ** (R2, R5, kéo theo R3): L2 vòng hở đẹp (0.626 m) nhưng khi tự lái, mô hình đi vào những trạng thái người lái không bao giờ đi vào (covariate shift), và tại đó nó không biết phải làm gì; tuỳ seed mà lỗi này nặng hay nhẹ.

Sửa: (a) thêm **196970 cửa sổ từ 20000 cảnh cắt vào mô phỏng**, nhãn là việc IDM làm ở đó, để chữa phần thiếu phủ kịch bản; (b) **2 vòng DAgger**: cho mô hình tự lái trên log train, ghi lại các trạng thái *nó tự đi vào*, gắn nhãn bằng dự báo 3 s của IDM *tại chính các trạng thái đó*, để chữa đúng phân phối trạng thái bị lệch (tổng ~894k cửa sổ). Sau khi sửa, PercMLP+ **qua cả 7 quy tắc**: R2 0.3 % (cận trên 0.6 %), R3 2.1 % (cận trên 5.2 %), R4 0.3 %, R5 0.3..0.3 %, R6 5.5 % so với 5.1 %. Đáng chú ý: L2 vòng hở **tệ đi** (0.626 → 0.830 m) trong khi va chạm giảm mạnh, thêm một bằng chứng rằng điểm vòng hở không phải thước đo an toàn.

PercMLP+ **không tốt hơn IDM**, chỉ **không kém hơn trong biên margin**: trên mọi số đo nó vẫn nhỉnh hơn IDM về va chạm (log 0.3 % so với 0.1 %; lát cắt phanh 2.1 % so với 1.0 %; lưới 0.3 % so với 0.0 %; thế giới phản ứng 5.5 % so với 5.1 %). Nó học nhãn **từ chính IDM**, nên về nguyên tắc chỉ xấp xỉ được thầy, không vượt được thầy. Ngoài miền huấn luyện (cruise 20 m/s, train chỉ thấy ≤ 16), PercMLP+ vẫn đâm 0.1..6.2 % lưới tránh được, trong khi IDM 0.0 %. Đề xuất: **chưa thay IDM**. Trước khi quyết định: chạy lái bóng (shadow mode) để so lệnh với IDM trên dữ liệu mới; thêm vào cổng các kịch bản ngoài miền (tốc độ cao, cắt ra, nhiễu cảm biến nặng hơn); và chỉ phát hành nếu có lợi ích ngoài an toàn (êm hơn, giống người hơn) đủ bù cho phần rủi ro còn lại.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

1. **Xe dẫn trong log không phản ứng với xe thử.** Ở bước 4, xe dẫn phát lại đúng quỹ đạo người lái, nên tương tác hai chiều (xe dẫn đổi hành vi vì xe thử bám sát) không được kiểm. Bước 5 đã cho thấy chỉ riêng việc xe sau không phản ứng đã tạo ra 64.9 % va chạm giả; điều ngược lại (bỏ sót va chạm thật) cũng có thể xảy ra ở phía trước.
2. **Thế giới mô phỏng là giả định của chúng ta**: một chiều, động lực học điểm, phanh tối đa 9 m/s², không trượt, không độ dốc, không thời tiết. Biên bất khả thi và mọi tỷ lệ va chạm đều đúng *trong mô hình này*. Mô hình học (và IDM) có thể vượt cổng ở đây nhưng hành xử khác trên xe thật với độ trễ cơ cấu chấp hành và mặt đường khác.
3. **Không có cảm biến thật.** Khoảng cách và tốc độ xe dẫn được đưa vào gần như hoàn hảo; phép thử nhiễu chỉ có gap ±1 m, tốc độ ±0.5 m/s (PercMLP 0.8..6.6 %). Không có mất dấu, phát hiện nhầm, hay trễ cảm biến biến thiên, vốn là nguồn lỗi chính của hệ thống thật.
4. **Độ phủ kịch bản hẹp**: dữ liệu Argoverse là phố đô thị (tốc độ p99 ~15.6 m/s), người lái phanh tối đa 3.6 m/s². Cổng chỉ kiểm cắt vào ở cruise 10.0 m/s và 614 ca phanh; không có tốc độ cao tốc, dừng-đi dày đặc, hay cắt ra để lộ xe đứng yên.

Việc cần làm tiếp: mô phỏng đa tác nhân với xe dẫn phản ứng, mô hình cảm biến có lỗi thực tế (dropout, false positive, trễ), mở rộng họ kịch bản (nhiều tốc độ cruise, cắt ra), kiểm thử phần cứng trong vòng (HIL) để đo hạn chót thời gian thật, rồi lái bóng (shadow mode) trên xe thật trước khi cho mô hình cầm lái.
