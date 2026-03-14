# 🧠 Logic & Thuật toán của CV Chess

Tài liệu này trình bày chi tiết về ý tưởng, thuật toán và sự liên kết giữa các file trong dự án CV Chess.

## 1. Phân bổ và Luồng chức năng ở các file mã nguồn

Dự án có 4 file chính với sự thay đổi tiếp cận trong quá trình phát triển:

- **`test_chess.py` (File hoàn thiện nhất):** Tổng hợp toàn bộ các tính năng nâng cao. Bao gồm:
  - Nhận diện tự động bằng Contour (Auto Mode).
  - Cho phép người dùng linh hoạt chuyển sang chế độ chọn 4 góc bằng tay ấn phím `m` (Manual Mode).
  - Tự động hiệu chuẩn lưới ô cờ không góc cạnh (Perspective Calibration) bằng HoughLines.
  - Ghi chuẩn PGN đầu ra.
- **`test_chess2.py` (OOP & Grid Preview):** Bản thử nghiệm thiết kế theo hướng đối tượng, tách logic thành class `ChessMoveDetector`. File này bổ sung thêm tính năng nhấn `g` để xem trước trực tiếp lưới phân vùng, tiện lợi cho việc setup góc nhìn của camera.
- **`test_chess3.py` (Auto Focus):** Bản tối giản hoàn toàn tự động bắt Contour, không có fall-back về tính năng Manual Corner. Là bản sao chép lược đoạn để độc lập hóa phần test thuật toán hình học mới.
- **`test_chess_manualConner.py` (Manual & Grid cố định):** Phiên bản sơ khai yêu cầu người dùng tuần tự click đủ 4 góc của bàn cờ (A8, H8, H1, A1). Lưới (Grid) chia đều 64 ô kích thước cố định thay vì Calibrate phức tạp. Rất ổn định khi camera đặt vuông góc mặt bàn.

---

## 2. Ý tưởng của các khối Thuật toán

### 2.1. Nhận diện bàn cờ & Cắt vùng nhìn (Board Detection & Perspective Warp)
**Mục tiêu:** Tìm ra tọa độ bàn cờ trong khung hình camera méo mó và biến đổi về mảng hiển thị 2D góc nhìn chính xác từ trên xuống (top-down view).

- **Tiền xử lý ảnh (thấy rõ tại `test_chess.py` & `test_chess3.py`):**
  - Đổi ảnh sang thang xám (`cv2.cvtColor`).
  - **Kỹ thuật Cân bằng sáng khu vực `CLAHE` (Contrast Limited Adaptive Histogram Equalization):** Giúp làm rõ các vị trí ô đen/trắng và vạch kẻ lưới bất chấp camera bị chói đèn hoặc bóng đen che khuất.
  - **Ngưỡng nhị phân hóa thông minh OTSU:** `cv2.threshold` kết hợp `cv2.THRESH_OTSU` tự động tìm mức cường độ sáng phân tách hợp lý nhất để cắt bạch bảng cờ và nền, thay vì bị hard-code một chỉ số fix.

- **Tìm kiếm Đa giác bao ngoài (Contour Analysis):**
  - Dùng thuật toán `cv2.findContours` tìm toàn bộ các hình thù khép kín. Giải thiết cái to nhất chỉnh là bàn cờ cần tìm. Tính viền bằng xấp xỉ đa giác `cv2.approxPolyDP`. Khi kết quả hội tụ về đúng một hình học có 4 đỉnh, ta coi đó là 4 góc bàn cờ ngoài cùng.

- **Perspective Transform (Làm phẳng quang học):**
  - Hàm `order_points` chuẩn hóa thứ tự 4 điểm đã bắt (Top-Left, Top-Right, Bottom-Right, Bottom-Left).
  - Gán vào `cv2.getPerspectiveTransform` để lấy ma trận làm phẳng `M`.
  - Hàm `cv2.warpPerspective` sẽ căng kéo bức ảnh chéo từ góc camera thành một bức ảnh phẳng hình vuông tỉ lệ 1:1, ví dụ size 500x500 pixels. Ở `test_chess.py`, ma trận `M` còn được lọc tần số thấp (Moving Average: `alpha * prev_M + (1-alpha) * M`) để không bị lung lay góc khi có nhiễu frame.

### 2.2. Hiệu chuẩn Lưới theo góc nhìn (Calibration - Trị méo thấu kính)
**Mục tiêu:** Do ống kính camera luôn có độ méo khối phố (perspective distortion), các ô vuông ở mép xa camera sẽ nhỏ hơn các ô ở gần. Việc chia mảng 500x500 ra 8 phần bằng nhau sẽ gây lệch trục rất lớn.

- Nhấn phím 'i', khung bàn cờ (đã được warp) được bắt viền bằng `cv2.Canny` và vẽ tiếp một loạt những đường thẳng trên đó bằng `cv2.HoughLines`.
- **Hàm `cluster_lines`:** Các đường kẻ đan xen bị xóa nhiễu và phân tách về hai mảng tọa độ ngang (`h_grid`) và dọc (`v_grid`). Kết quả lý tưởng phải trả về đủ 9 tọa độ dọc (bao gồm 2 viền ngoài và 7 vạch trong) và 9 tọa độ ngang.
- Thông qua bộ mốc `h_grid[r]`, `v_grid[c]`, hệ thống tạo ra một tấm thảm có kích thước không đều, uốn chuẩn vào mọi ô riêng lẻ trong khung ảnh. 

### 2.3. Trích xuất khác biệt - Phát hiện nước đi (Move Detection)
**Mục tiêu:** Nhận biết tay người đang di chuyển những cờ gì, nhấc khỏi ô nào và hạ xuống ô nào.

- **Trừ trừ ảnh động (Absolute Difference):**
  - Hàm `cv2.absdiff` tính trị tuyệt đối sự khác nhau từng pixel giữa 2 tấm ảnh (Tấm ảnh A: Lưu ở lần đi cờ ngay trước đó so với Tấm ảnh B: Tại thời điểm người dùng nhấn SPACE).
  - Kết quả ra một bức ảnh tối đen, nhưng vùng có tay hoặc sự di chuyển cờ sẽ trắng sáng.
  - Dành qua ngưỡng lọc `cv2.threshold` để loại bỏ nhiễu hạt bụi.

- **Tính toán lượng di chuyển dựa theo ô lưới:**
  - Vòng lặp duyệt r=0..7, c=0..7 lấy cắt chính xác phần diện tích theo `h_grid`, `v_grid` của từng ô.
  - Hàm `cv2.countNonZero(roi)` đếm tổng lượng pixel "sự kiện" ở ô đó. 
  - Nếu số pixel sinh ra đạt ngưỡng yêu cầu (dùng để bỏ đi thay đổi vi tế, vd > 100 px), gắn nhãn diện tích ô đó bị thay đổi hình ảnh và đưa vào mảng sort giảm dần.
  - Hai ô có pixel thay đổi nhiều nhất sẽ là đối tượng suy luận cao nhất, vì thực tế sự chuyển đổi ảnh hưởng rõ ràng nhất là việc "rỗng quân ở vị trí xuất phát" và "đè hình quân lên ở vị trí đến".

### 2.4. Khớp Logic Cờ (Chess Inference với python-chess)
**Mục tiêu:** Một hình ảnh nhiễu không thể phỏng đoán nước đi nếu thiếu Luật của game. Dự án dùng thư viện `python-chess` làm trọng tài bù trừ.

- Từ mảng những ô có sự thay đổi từ trên xuống dưới, hệ thống nhặt ra danh sách 4 ô đầu. (Vì sao lấy 4? Vì trường hợp nhập thành - Castling sẽ tác động 4 biến đối, Ăn quân - Capture chỉ 2, Đi thường có thể 2 hoặc 3 nếu tay che bóng).
- Thư viện `python-chess` kiểm tra board hiện tại có thể tạo ra mảng `legal_moves` (các bước đi có thể xảy ra theo luật quốc tế).
- Thuật toán `infer_move` chạy vòng lặp xem các nước đi nào có `from_square` (tọa điểm khởi xuất) & `to_square` (trạm đáp trả) đồng bộ với nằm trong nhóm 4 ô thay đổi mạnh nhất.
- Khi chắt lọc được 1 kết quả cụ thể, nước đi được Push vào Virtual Board. Diễn tiến được cập nhật vào `game_recorded.pgn` dưới định dạng chuỗi SAN, ví dụ: 1. e4 e5 2. Nf3. Nếu xảy ra mâu thuẫn hình ảnh (vài biến đổi quá phức tạp cản trở phân tích) hàm trả về Warning Ambiguous hoặc Cảnh báo không tìm thấy Nước Legal Move.
