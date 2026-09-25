# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: `Nguyễn Văn Hoàng`

Công cụ gán nhãn đã dùng: `CVAT` (AnyLabeling, CVAT, SAM hoặc sửa trực tiếp file nhãn)


## 1. Dữ liệu và cách chia tập

Tại sao tập chưa gán nhãn (pool) và tập kiểm thử (test set) được chia theo trục thời gian, có vùng
đệm ở giữa, thay vì chia ngẫu nhiên? Nếu chia ngẫu nhiên, số đo trên tập kiểm thử sẽ bị lệch theo
hướng nào, và vì sao?

`Camera đứng một chỗ, một chiếc xe nằm trong hình vài giây, nên các frame sát nhau về thời gian gần như là cùng một cảnh. Vì vậy ảnh học (pool) và ảnh kiểm tra (test) phải cách nhau theo thời gian, có vùng đệm ở giữa. Nếu trộn ngẫu nhiên, cùng một chiếc xe có thể vừa được model học ở ảnh này vừa được dùng để chấm ở ảnh bên cạnh. Khi đó điểm đo được sẽ đẹp hơn sự thật: số đo bị lệch theo hướng lạc quan, vì model đã thấy trước đáp án.`

## 2. Mô hình khởi đầu lạnh (cold start)

Chép dòng vòng 0 từ `rounds_table.md`. Dựa vào `outputs/compare_round0.jpg`, cho biết mô hình khởi
đầu lạnh không khớp nhãn tham chiếu ở những loại xe nào. Độ phủ (recall) theo kích thước xe cho
thấy điều gì? Một trường hợp nào cần người rà lại nhãn tham chiếu trước khi kết luận mô hình sai?

`Dòng vòng 0 trong rounds_table.md: yolov8n cold start (COCO car+bus+truck), 0 ảnh train, AP50 0.771, precision 0.925, recall 0.489, F1 0.640. Recall theo cỡ xe là small 0.182, medium 0.547, large 0.561. Nghĩa là xe ở xa (nhỏ) bị bỏ sót nhiều nhất, xe ở gần được tìm thấy tốt hơn nhưng cũng chỉ hơn một nửa.`

`Trong compare_round0.jpg, ở frame_0350 nhãn tham chiếu có 23 box nhưng model chỉ khớp 9 (TP 9, FP 2, FN 14). Cụm xe nhỏ gần đầu đường, chỉ còn là vài chấm đèn, bị bỏ sót gần hết, và có một box đỏ (box thừa) vẽ trùm lên cả cụm xe bên trái. Ở frame_0050 một chiếc xe lớn ở góc dưới bên trái cũng bị bỏ sót.`

`Nhãn dùng để chấm cũng do máy vẽ, chưa có người xem từng box, nên có thể nhãn chấm sai chứ không phải model sai. Ví dụ ở frame_0350, box đỏ ở cụm xe đầu đường cần người nhìn lại: nếu trong cụm đó có xe thật mà nhãn tham chiếu không vẽ, thì đó là lỗi nhãn chứ không phải lỗi của model.`

## 3. Chiến lược chọn mẫu

Giải thích bằng lời công thức `score = W_U·U + W_A·A + W_D·D` và vai trò của `MIN_GAP_S`.
Dẫn ba frame trong `reports/SELECTION.md` và một frame khác để chứng minh cách bạn cân nhắc
độ bất định, ảnh gần trùng và công gán nhãn. Điểm bất định có chứng minh ảnh đó sẽ cải thiện
mô hình không? Vì sao?

`Mỗi ảnh có một điểm score = 0.5·U + 0.3·A + 0.2·D. Một nửa điểm (U) là model không chắc: lấy 5 box khó nhất của ảnh, box nào có độ tin cậy gần 0.5 thì model phân vân nhất. Ba phần mười (A) là model vẽ nhiều box còn lưỡng lự, tức số box có độ tin cậy từ 0.15 đến dưới 0.50, chia cho ảnh có số box như vậy nhiều nhất trong pool. Hai phần mười (D) là ảnh có khác thời gian với các ảnh đã gán nhãn hay không; ở vòng 1 chưa có ảnh nào đã gán nên D bằng 1 cho mọi ảnh. MIN_GAP_S là 2 giây: hai ảnh trong cùng lô phải cách nhau ít nhất 2 giây, vì camera đứng yên, ảnh sát nhau gần như giống hệt và gán cả hai chỉ tốn công.`

`Ba ảnh tôi đã nêu trong SELECTION.md là frame_0182.jpg (hạng 1, điểm 0.959, 28 box trong đó 18 box còn lưỡng lự), frame_0099.jpg (điểm 0.906, 14 box lưỡng lự) và frame_0107.jpg (điểm 0.888, 15 box lưỡng lự). Cả ba đều nằm trong lô 12 ảnh. Ảnh còn lại là frame_0372.jpg, hạng 6, điểm 0.910, cao hơn vài ảnh được chọn nhưng bị bỏ vì cách frame_0369.jpg chỉ 1.2 giây (148.8 so với 147.6), hai ảnh gần như một cảnh. Về công gán nhãn, frame_0369.jpg model chỉ đề xuất 14 box mà sau khi sửa còn 35 box (phải thêm 22 box), nên mỗi ảnh khó như vậy tốn khá nhiều thời gian.`

`Điểm cao không có nghĩa sửa ảnh đó sẽ làm model giỏi hơn. Điểm chỉ cho biết model đang phân vân ở ảnh đó, chứ chưa chứng minh sửa xong thì model sẽ nhận xe tốt hơn.`

## 4. Các vòng học chủ động (active learning)

Chép bảng từ `rounds_table.md`. Với mỗi vòng, trình bày:

- mức độ bạn đã sửa nhãn gợi ý (số box giữ nguyên, chỉnh sửa, xoá, thêm mới, lấy từ
  `outputs/round*_diff.md`);
- AP50 thay đổi bao nhiêu so với khởi đầu lạnh và so với vòng trước;
- nhóm xe nào tốt lên hoặc xấu đi theo số đo trên cùng tập test.

Dựa vào các ảnh `compare_round*.jpg`, chỉ ra một ca kết quả đổi sau fine-tune (tốt hơn hoặc xấu
đi), cùng lý do có thể kiểm. Dùng `BLIND_SCAN.md`, `REVIEW_LOG.csv` và `round1_diff.md` phân biệt
quan sát độc lập, lỗi pre-label đã sửa và kết quả mô hình sau train. Mô tả một ca khó theo guideline.

`Bảng sao chép từ rounds_table.md (tập kiểm thử 20 ảnh, 403 box tham chiếu, bỏ qua 14 box cao dưới 16 px, IoU 0.5, P/R/F1 tính tại conf 0.25):`

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vong 1..1 | 12 | 327 | 0.805 | +0.034 | 1.000 | 0.246 | 0.394 | 0.000 | 0.240 | 0.683 |


`Vòng 1: theo round1_diff.md, model đề xuất 169 box cho 12 ảnh. Tôi giữ nguyên 106 box, chỉnh sửa 45 box, xóa 18 box (model vẽ thừa) và thêm mới 176 box (model bỏ sót), còn lại 327 box; tỷ lệ giữ nguyên là 63%. AP50 tăng từ 0.771 lên 0.805, tức +0.034 so với cold start. Nhưng recall tại conf 0.25 lại giảm từ 0.489 xuống 0.246, và precision tăng từ 0.925 lên 1.000 (FP từ 16 xuống 0, TP từ 197 xuống 99). Theo nhóm xe: xe lớn tốt lên (0.561 lên 0.683), xe vừa xấu đi (0.547 xuống 0.240), xe nhỏ xấu đi (0.182 xuống 0.000).`

`Một ca đổi sau fine-tune, xem compare_round0.jpg và compare_round1.jpg. Ở frame_0350 chiếc xe lớn ở góc dưới bên phải trước đây bị bỏ sót nay đã được khớp, phù hợp với việc recall xe lớn tăng. Ngược lại ở frame_0050 mấy xe nhỏ phía trên bên phải trước đây khớp nay bị bỏ sót, và tổng khớp của ảnh giảm từ 11 xuống 7 (frame_0350 giảm từ 9 xuống 6). Lý do có thể kiểm: FP bằng 0 nhưng recall thấp cho thấy model có vẻ chỉ còn dám báo xe khi rất chắc. Đây mới là suy đoán, tôi chưa kiểm tra, nên cần chạy lại với ngưỡng conf thấp hơn 0.25 để xem có phải các box nhỏ chỉ bị điểm tin cậy thấp hay không.`

`Ba việc khác nhau. Một, quan sát độc lập của mắt tôi: trong BLIND_SCAN.md, ở frame_0099.jpg tôi tự đếm 26 xe trước khi xem gợi ý và ghi hai chỗ model dễ sai (một xe bị khung hình cắt mất khoảng một nửa ở góc dưới bên phải, và hai xe đi sát nhau, một xe bị xe kia che khuất). Hai, lỗi của pre-label mà tôi đã sửa: ghi trong REVIEW_LOG.csv, gồm frame_0099.jpg (thêm box cho xe thứ hai trong cặp xe sát nhau vì model chỉ vẽ một box), frame_0107.jpg (thêm box cho xe bị cắt ở mép dưới ảnh) và frame_0392.jpg (xóa box xe buýt mà model nhầm là car). Ba, kết quả của model sau khi học lại: là các số ở bảng trên và ảnh compare_round1.jpg, không phải việc tôi tự thấy hay tự sửa.`

`Ca khó theo guideline: xe bị cắt bởi mép ảnh ở frame_0107.jpg vẫn còn nhìn thấy một phần thân xe nên tôi vẫn vẽ box quanh phần nhìn thấy; frame_0099.jpg cũng có ca hai xe sát nhau, xe bị che một phần vẫn được vẽ box riêng thay vì gộp thành một box.`

## 5. Kết luận và giới hạn

Kết quả vòng này so với cold start ra sao? Vì sao bạn dừng hoặc tiếp tục? Đề xuất hai ca còn yếu
hoặc bất định cho vòng sau, kèm chi phí rà nhãn và nguy cơ ảnh gần trùng. Tập kiểm thử chỉ 20 ảnh,
có luật bỏ qua xe quá nhỏ và nhãn tham chiếu do mô hình tạo chưa được rà thủ công; các giới hạn đó
ảnh hưởng thế nào đến kết luận? Nếu AP50 giảm, bạn sẽ kiểm tra điều gì trước khi train thêm?

`Điểm vòng 1 là AP50 0.805, cao hơn cold start 0.771 một khoảng +0.034, nhưng recall tại conf 0.25 giảm từ 0.489 xuống 0.246 và recall xe nhỏ về 0.000. Vì vậy tôi chọn làm tiếp thêm một vòng, chưa dừng: AP50 chỉ tăng nhẹ, xe nhỏ và xe vừa còn rất yếu, và mới học từ 12 ảnh. Trước vòng sau tôi sẽ chạy lại phần chấm với ngưỡng conf thấp hơn để biết recall giảm vì model thật sự bỏ sót hay chỉ vì điểm tin cậy tụt dưới 0.25.`

`Hai chỗ còn yếu: một là xe ở xa chỉ còn hai chấm đèn (như cụm xe đầu đường ở frame_0350), recall xe nhỏ đang bằng 0; hai là xe bị cắt mép hoặc bị xe khác che (như ở frame_0107 và frame_0099). Sửa thêm thì mất thời gian: mỗi ảnh vòng 1 tôi phải thêm khoảng 10 đến 22 box. Cũng không nên chọn hai ảnh sát nhau, ví dụ frame_0368, frame_0369 và frame_0372 cách nhau chưa đến 2 giây, vì chúng gần như một cảnh và sửa cả ba là tốn công mà ít học thêm được gì.`

`Giới hạn: phần chấm chỉ có 20 ảnh, xe quá nhỏ (cao dưới 16 px, 14 box) không tính, và nhãn chấm do máy vẽ chưa được người kiểm, nên mức tăng +0.034 chưa đủ để kết luận chắc, và có thể nhãn chấm sai chứ không phải model sai. Nếu điểm giảm, tôi sẽ xem lại các box đã sửa trong REVIEW_LOG.csv trước khi cho model học thêm.`
