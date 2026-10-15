MSSV: 202602579

# Lab 11 · Scenario Forge — báo cáo

Mỗi câu: trả lời ngắn, **trích số từ `evidence/`** (ghi tên file và trường, ví dụ `mc.json: crashes = 4`). Grader chạy lại với MSSV của bạn; số không khớp `evidence/` thì mất điểm phần D (RUBRIC), và câu dựa trên số sai khó được điểm đầy đủ. Không cần văn hay: một bảng nhỏ và hai, ba câu giải thích là đủ.

Thị trấn của bạn (`meta.json: _run.town`): A

## 1. Phát lại log có trung thực không? (bước A)

Theo `replay.json`:
- Số kịch bản phát lại: `n = 400`, có `dropped_human_crash_logs = 1` kịch bản bị bỏ vì người lái thật gây va chạm, số kịch bản dùng thực tế `n_used = 399`.
- Va chạm trên log phát lại (`replay.crashes`): **121** va chạm, gồm `contact.rear = 104` (86%), `contact.side = 17` (14%), `contact.front = 0`.
- Va chạm trong thế giới phản ứng (`reactive.crashes`): **0** va chạm (`contact = {}`), `crash_in_both = 0`.

Phần lớn va chạm của "phát lại" là `rear` và đây là **va chạm giả** (*spurious collision*): Trong log phát lại, quỹ đạo của các xe khác bị đóng băng theo diễn biến lúc người lái thật điều khiển. Khi xe ta (ego) chạy planner tự động, planner phanh sớm hoặc thận trọng hơn người lái thật. Xe bám đuôi (TG) trong log phát lại không biết xe ta phanh sớm nên vẫn lao tới theo tốc độ cũ và đâm vào đuôi xe ta. Trong thế giới phản ứng, TG lái theo IDM nên khi xe ta phanh, TG cũng phanh nhường theo và không hề có va chạm (`reactive.crashes = 0`).

Ví dụ kịch bản `replay_only[0]` (Scenario 5): Va chạm loại `rear`, trong khi ở thế giới phản ứng thì `min_ttc_reactive = 2.158` s (rất an toàn). Xe cắt làn C2 nhập làn lúc $t = 1.91$ s và phanh với gia tốc $3.32\text{ m/s}^2$; xe ta phanh giảm tốc để nhường đường, xe TG trong log cũ không phản ứng với cú phanh này nên đã đâm vào đuôi xe ta.

## 2. Kịch bản bất khả thi (bước B)

Bốn luật trong `is_feasible` và căn cứ vật lý:
1. Ma sát ngang khi chuyển làn $a_{\text{lat}} = (\pi / \text{cutDur})^2 \cdot 3.5 / 2 \le \mu g$: Xuất phát từ quỹ đạo chuyển làn cosin ngang 3.5 m. Nếu gia tốc ngang đỉnh vượt quá lực bám cực đại $\mu g$, xe sẽ bị trượt văng đuôi mất lái (*spin-out*).
2. Phanh sau nhập làn $p[\text{"cutBrake"}] \le \mu g$: Giới hạn ma sát trượt của lốp xe trên mặt đường ($F_{\text{phanh}} \le \mu m g \implies a \le \mu g$). Phanh mạnh hơn $\mu g$ là phi thực tế vì bánh xe sẽ bị bó cứng trượt lết.
3. Tốc độ người đi bộ $p[\text{"pedV"}] \le 2.5\text{ m/s}$: Tương đương $9\text{ km/h}$, là tốc độ chạy nước rút tối đa của người đi bộ bình thường qua đường đô thị. Vận tốc cao hơn thuộc về vận động viên chuyên nghiệp.
4. Tốc độ xe cắt làn $p[\text{"cutV"}] \in (0, 22]\text{ m/s}$: Xe phải tiến về phía trước ($> 0$) và không vượt quá $22\text{ m/s}$ ($\approx 79.2\text{ km/h}$), là trần tốc độ cho phép của tuyến đường đô thị mô phỏng.

Trong `hunt.json`, số kịch bản bất khả thi bị loại bỏ:
- `hunt rear_ended`: **10** kịch bản bất khả thi.
- `hunt crosswalk_block`: **6** kịch bản bất khả thi.
- `hunt cut_in`: **4** kịch bản bất khả thi.
- `hunt bus_bypass`: **1** kịch bản bất khả thi.

Cuộc săn `rear_ended` sinh nhiều kịch bản bất khả thi nhất vì thuật toán CEM bị kéo về các tình huống phanh/chuyển làn cực đoan (gia tốc ngang cực lớn hoặc phanh phi vật lý), vốn tạo ra TTC sát nút nhất. Nếu **đếm** chúng là va chạm, ước lượng xác suất rủi ro của planner sẽ bị thổi phồng giả tạo (chệch dương nghiêm trọng), báo cáo những nguy cơ mà trên thực tế xe cộ không thể tạo ra được.

## 3. Monte Carlo cho bạn biết gì (bước C1)

Trích xuất từ `mc.json`:
- $n = 3000$, $k = 7$ va chạm.
- $\hat{p} = 0.002333$ ($2.33 \times 10^{-3}$).
- Khoảng tin cậy Wilson 95%: $[0.001131, 0.004809]$ (tức $[1.13 \times 10^{-3}, 4.81 \times 10^{-3}]$).

Cỡ mẫu cần thiết để khoảng tin cậy hẹp lại còn $\pm 20\%$ của $p$:
$$\frac{z \cdot \sqrt{p(1-p)/n}}{p} \le 0.2 \implies n \approx \frac{z^2(1-p)}{0.04 \cdot p} \approx \frac{96.04}{p}$$
Với $\hat{p} \approx 0.002333$:
$$n \approx \frac{96.04}{0.002333} \approx 41,160\text{ lần chạy.}$$

Các kiểu thất bại **chưa hề xuất hiện** trong 3.000 lần chạy ($k = 0$): `cut_in`, `crosswalk_block`, `rear_ended`, `other`. Toàn bộ 7 vụ va chạm đều thuộc kiểu `bus_bypass`.
Con số đúng cần báo cho các kiểu chưa thấy: Báo cận trên của khoảng tin cậy Wilson 95%: `mc.json: per_mode.<mode>.hi = 0.001279` ($1.28 \times 10^{-3}$) (hoặc quy tắc số ba: $3/n = 3/3000 = 1.0 \times 10^{-3}$), **tuyệt đối không báo bằng 0**.

## 4. Importance sampling, ESS và độ suy biến (bước C3)

Bảng tổng hợp từ `estimate.json`:

| Kiểu thất bại | Seed | $p$ | hits | ess | top_share |
|---|---|---|---|---|---|
| `bus_bypass` | 0 | $1.82 \times 10^{-3}$ | 30 | 136.3 | 0.349 |
| `bus_bypass` | 1 | $8.34 \times 10^{-4}$ | 27 | 122.4 | 0.194 |
| `bus_bypass` | 2 | $7.53 \times 10^{-4}$ | 31 | 118.5 | 0.159 |
| `crosswalk_block` | 0 | $1.03 \times 10^{-4}$ | 52 | 99.1 | 0.947 |
| `crosswalk_block` | 1 | $1.31 \times 10^{-5}$ | 57 | 123.5 | 0.474 |
| `crosswalk_block` | 2 | $1.91 \times 10^{-6}$ | 45 | 131.4 | 0.332 |
| `cut_in` | 0, 1, 2 | 0.0 (cận: 0.025) | 0 | 111.5 – 139.6 | 0.0 |
| `rear_ended` | 0, 1, 2 | 0.0 (cận: 0.025) | 0 | 122.8 – 145.4 | 0.0 |
| **Total** | 0 | $1.92 \times 10^{-3}$ | — | — | — |
| **Total** | 1 | $8.47 \times 10^{-4}$ | — | — | — |
| **Total** | 2 | $7.55 \times 10^{-4}$ | — | — | — |

- **Đồng thuận giữa 3 seed và với Monte Carlo:**
  - Với `bus_bypass`, cả 3 seed đều cho kết quả cùng bậc độ lớn ($7.53 \times 10^{-4}$ đến $1.82 \times 10^{-3}$), nằm hoàn toàn trong khoảng tin cậy của Monte Carlo $[1.13 \times 10^{-3}, 4.81 \times 10^{-3}]$.
  - Với `crosswalk_block`, 3 seed chênh nhau đáng kể (từ $1.91 \times 10^{-6}$ ở seed 2 đến $1.03 \times 10^{-4}$ ở seed 0).
- **Vị trí `top_share` lớn (> 0.5):**
  - Xuất hiện tại `crosswalk_block` ở Seed 0 với `top_share = 0.947` (94.7%).
  - Khi đó, phân phối đề xuất $q$ co vào vùng hiếm, một mẫu đơn lẻ có trọng số $w = p/q$ cực lớn chi phối tới $94.7\%$ toàn bộ giá trị ước lượng (suy biến trọng số - *weight degeneracy*). Con số ước lượng khi đó thực chất chỉ là giá trị của mẫu đơn lẻ đó.
- **So sánh ngân sách (`estimate.json: budget = 16800`):**
  - Để Monte Carlo bắt được sự kiện hiếm cỡ $10^{-5}$ đến $10^{-6}$ với cùng độ chính xác $\pm 20\%$, cần $n \approx 96/p \approx 9.6 \times 10^6$ đến gần 100 triệu lần chạy. IS đạt được độ phân giải này chỉ với 16.800 lần chạy (tiết kiệm hàng trăm đến hàng ngàn lần chi phí tính toán).
- **Phần `other`:**
  - Lấy từ `mc.json`: `k = 0, n = 3000, hi = 0.001279`.
  - Không săn `other` vì đây là nhóm va chạm không định hình, không có hàm điểm nguy hiểm (*danger score*) đặc thù để hướng dẫn thuật toán CEM co cụm phân phối đề xuất.

## 5. Các kiểu thất bại (bước C2 + 3D)

Hai kiểu thất bại được tìm thấy:
1. **`bus_bypass`**:
   - Ảnh chụp: `evidence/shots/bus_bypass.jpg`
   - Chuỗi sự kiện: Xe buýt đỗ ở làn ngoài che khuất tầm nhìn; xe C2 lách qua xe buýt với vận tốc chậm; xe ta tiến tới không thấy trước được C2 nên không kịp giảm tốc, dẫn đến va chạm trực diện/chớm góc tại đầu xe buýt.
2. **`crosswalk_block`**:
   - Ảnh chụp: `evidence/shots/crosswalk_block.jpg`
   - Chuỗi sự kiện: Người đi bộ P1 bước qua đường từ phía sau xe buýt đỗ; tầm nhìn bị xe buýt chắn; khi xe ta phát hiện ra P1 thì khoảng cách quá gần nên phải phanh gấp, xe dừng hẳn nhưng mũi xe chớm đè lên vạch qua đường, khiến người đi bộ phải đứng lại (`kind == "block"`).

Kiểu cuộc săn **không** tìm thấy: `cut_in` và `rear_ended`.
- Mức độ tin cậy: Rất cao, vì IDM của xe bám đuôi TG có biên an toàn và khả năng phanh $b = 3.0\text{ m/s}^2$ đủ nhạy để thích ứng với cú phanh của xe ta; còn xe cắt làn C2 có cơ chế gap acceptance nên chỉ chuyển làn khi có khoảng trống hợp lý.
- Nếu chỉ chạy **một** cuộc săn chung: Thuật toán CEM sẽ bị co cụm phân phối (*mode collapse*) vào kiểu va chạm có gradient điểm nguy hiểm dốc và dễ xảy ra nhất là `bus_bypass` (như thấy trong `hunt.json`, cuộc săn `cut_in` cũng toàn tìm ra `bus_bypass: 29`), bỏ sót hoàn toàn các kiểu va chạm hiếm gặp hơn như `crosswalk_block`.

## 6. Sửa planner (bước E + 3D)

Trích xuất từ `fix.json`:
- Policy được chọn: `{'cutinLat': 3.0}` (mở rộng ngưỡng ngang nhận diện xe cắt làn từ 2.15 m lên 3.0 m).
- Trên **TRAIN**:
  - `chosen`: `fail = 3.26e-06`, `fail_hi = 7.33e-06`, `unseen = ['bus_bypass']`, `progress = 210.7 m`.
  - `baseline`: `fail = 4.23e-04`, `fail_hi = 7.69e-04`, `progress = 210.9 m`, `harsh = 8.82`.
- Trên **TEST**:
  - `chosen`: `fail = 1.81e-07`, `fail_hi_seen = 3.96e-07`, `fail_hi = 3.88e-02`, `unseen = ['bus_bypass']`, `progress = 214.6 m`.
  - `baseline`: `fail = 6.75e-04`, `fail_hi_seen = 1.30e-03`, `fail_hi = 2.58e-03`, `unseen = []`, `progress = 214.6 m`.
- Quãng đường trung bình: `progress = 214.6 m` (giữ nguyên so với baseline, không mất quãng đường).

**Vì sao `fail_hi` của bản sửa lớn hơn `fail` của baseline dù bản sửa tốt hơn thật:**
Bản sửa triệt tiêu hoàn toàn va chạm `bus_bypass` trên TEST, khiến kiểu này trở thành `unseen`. Về mặt thống kê, với $n = 400$ mẫu TEST và defensive mixture $0.2$, quy tắc Rule of Three áp dụng cận trên bảo thủ 95% cho kiểu unseen là $\text{upper\_zero\_hits} = 3 / (0.2 \times 400) = 0.0375$. Grader cộng cận bảo thủ này vào `fail_hi` ($1.81 \times 10^{-7} + 0.0375 + \dots = 0.0388$). Đây là sự trung thực về mặt thống kê: không thể chứng nhận một kiểu va chạm đã biến mất dưới $10^{-2}$ nếu chỉ kiểm tra 400 lần chạy!

**Ảnh 3D so sánh với bóng mờ:**
Ảnh chụp: `evidence/shots/fix_ghost.jpg` (hoặc `fix_ghost.png`: xe chính chạy chính sách gốc va chạm với C2 lúc 12.6 s, trong khi bóng mờ là bản sửa với `cutinLat: 3.0` nhận diện C2 sớm hơn và dừng lại an toàn ở phía sau).

**Núm chỉnh "nghe thì an toàn hơn" nhưng thực tế làm tăng thất bại:**
Núm `cautionV`: Giảm `cautionV` từ 7.5 m/s xuống 5.0 m/s (chạy rất chậm khi qua vùng vạch kẻ bị che khuất). Khi xe ta đi quá chậm gần xe buýt, thời gian xe ta chiếm dụng vùng xung đột kéo dài hơn gấp đôi. Xe cắt làn C2 phán đoán sai hoặc thiếu kiên nhẫn vẫn lách ra và va chạm vào thân xe ta, đồng thời làm quãng đường xe đi được sụt giảm nghiêm trọng (`progress` giảm từ 210.9 m xuống 171.2 m, mất gần 19% tiến độ).

## 7. Từ mô phỏng ra đường thật

Ba giả định ảnh hưởng nhiều nhất trong `space.py`:
1. **Khe chấp nhận của xe chuyển làn (`cutGap`)**:
   - *Dữ liệu thật:* Quỹ đạo thực tế từ camera flycam trên cao (dataset highD, NGSIM) hoặc dữ liệu telemetry đội xe tự hành ghi lại hàng ngàn ca cut-in tự nhiên.
   - *Hướng sai lệch:* Nếu sim giả định `cutGap` nhỏ hơn thực tế (cho rằng tài xế liều lĩnh hơn), tỷ lệ va chạm cut-in/bus-bypass trong sim sẽ cao hơn thực tế (quá bi quan).
2. **Tỷ lệ người đi bộ ngó qua xe buýt (`pedLook`)**:
   - *Dữ liệu thật:* Dữ liệu camera perception và eye-tracking hành vi người đi bộ đô thị (dataset JAAD, PIE).
   - *Hướng sai lệch:* Nếu sim giả định người đi bộ ít ngó xe hơn thực tế, xác suất `crosswalk_block` trong sim sẽ cao hơn thực tế, ép planner phải phanh gấp không cần thiết.
3. **Hệ số ma sát đường ướt ($\mu = 0.5$ vs khô $0.8$)**:
   - *Dữ liệu thật:* Dữ liệu đo lực bám mặt đường thực tế từ cơ quan quản lý giao thông và cảm biến độ trượt bánh xe ESP/ABS thực tế theo mùa thời tiết.
   - *Hướng sai lệch:* Nếu sim giả định ma sát không xét đến các điểm trơn trượt cục bộ ($\mu < 0.3$), quãng đường phanh tính toán sẽ ngắn hơn thực tế, dẫn đến nguy cơ va chạm ngoài đời thực cao hơn sim dự báo.

## 8. Giới hạn

Ba điều mà xác suất báo cáo **không** nói:
1. **Hình học giao thông đơn điệu:** Thế giới chỉ có một đoạn phố thẳng với 2 làn và một vạch qua đường; không có ngã tư có đèn tín hiệu, vòng xuyến, xe máy luồn lách hay vật cản tĩnh trên lòng đường.
2. **Hành vi tác nhân nhân tạo:** Các xe khác chỉ tuân thủ mô hình vi mô IDM đơn giản, người đi bộ chỉ đi thẳng với vận tốc cố định; không có hành vi giao tiếp con người (bấm còi, nháy đèn, vẫy tay ra hiệu) hoặc tài xế mất tập trung/say xỉn.
3. **Cảm biến lý tưởng:** Cảm biến trên xe ta nhận diện hoàn hảo 100% các đối tượng (bounding box chuẩn xác, không trễ, không false positive/negative, không bị nhiễu mưa/bụi), chỉ bị giới hạn bởi hình học che khuất của xe buýt.
- **Nếu được thêm một tuần:** Sẽ đo lường ảnh hưởng của độ trễ cảm biến (sensor latency 100–300 ms) và tracking jitter, đồng thời mở rộng mô phỏng cho nút giao thông phức tạp với sự xuất hiện của xe 2 bánh.

## 9. Rắc rối của bạn (bước G + chương 10)

- **Rắc rối:** Xe dẫn đường (L1) phanh khẩn cấp đột ngột ngay phía trước xe ta trên làn E1 để tránh chướng ngại vật bất ngờ.
- **Căn cứ:** Xe ta bám đuôi theo mô hình IDM. Khi xe trước phanh gấp với gia tốc lớn ($> 6\text{ m/s}^2$) ở khoảng cách gần, planner có thể bị bất ngờ nếu khoảng cách thời gian $T$ hoặc khả năng hãm phanh không đủ.
- **Giả định (`hazard.json: why`):**
  - `space`: `h0_decel` $[4.5, 7.2]\text{ m/s}^2$, `h0_dur` $[1.5, 2.8]\text{ s}$, `h0_trig` $[14.0, 26.0]\text{ m}$.
  - `exposure`: $0.04$ (4% các chuyến đi gặp tình huống xe trước phanh gấp, theo thống kê sự kiện tới hạn từ nghiên cứu NHTSA 100-Car Naturalistic Driving Study).
- **Kết quả** (theo `mine.json`):
  - Tỷ lệ khả thi: `accepted = 0.83` (83% - 1.245 / 1.500 kịch bản khả thi).
  - Baseline: 2 va chạm, gồm `avoidable = 0`, `unavoidable = 0`, `unattributed = 2`, `low_speed = 0`.
  - Chosen: 0 va chạm (`avoidable = 0`, `unavoidable = 0`, `unattributed = 0`).
  - $P(\text{lỗi planner} \mid \text{rắc rối}) = 0.00$ kèm khoảng Wilson 95%: $[2.17 \times 10^{-19}, 3.08 \times 10^{-3}]$.
  - Tần suất mỗi chuyến: $0.00$ (cận trên $1.23 \times 10^{-4}$ mỗi chuyến), nhỏ hơn nhiều so với tần suất lách xe buýt $8.34 \times 10^{-4}$.
  - *Giải thích:* Planner xử lý rất tốt tình huống này nhờ mô hình IDM liên tục theo dõi vận tốc và khoảng cách tới xe dẫn đầu L1, chủ động giảm tốc mượt mà giữ khoảng cách an toàn $s_0 + vT$ nên không xảy ra va chạm nào do lỗi planner.
- **Hình:** Ảnh chụp giao diện Studio/chương 10: `evidence/shots/hazard_studio.jpg`.
- **Bạn tự lái thì sao:** Khi tự lái, con người nhìn thấy đèn phanh của L1 sáng đỏ từ xa nên chủ động nhả chân ga và rà phanh sớm hơn 0.5–1.0 s so với IDM, giúp xe dừng lại êm ái hơn và không bị giật mình.
- **Vì sao "tránh được" là chuẩn lạc quan:**
  Phép đo oracle braking kích hoạt phanh tối đa ở giới hạn bám của lốp ($\mu g$) ngay từ khung hình đầu tiên rắc rối xuất hiện mà không chịu bất kỳ độ trễ nhận thức hay cơ cấu chấp hành nào, do đó nó đếm **nhiều hơn** số va chạm một planner thật có thể tránh được (chuẩn lạc quan). Ngược lại, oracle chỉ thử duy nhất phương án phanh thẳng mà không xét khả năng đánh lái (evasive maneuver) sang làn trống, khiến nó có thể đếm **ít đi** trong các tình huống lách xe an toàn.
- **Cách sửa:**
  Dù planner hiện tại đã kiểm soát tốt tình huống này, để tối ưu hơn nữa có thể bổ sung logic nhận diện tín hiệu đèn phanh xe phía trước (brake light detection) để kích hoạt phanh sớm hơn 0.3 s, hoặc tự động nâng thông số headway time $T$ từ 1.6 s lên 1.9 s khi hệ số ma sát mặt đường $\mu < 0.65$.

