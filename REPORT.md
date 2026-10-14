MSSV: 2202602451

# Lab 11 · Scenario Forge — báo cáo

Mỗi câu: trả lời ngắn, **trích số từ `evidence/`** (ghi tên file và trường, ví dụ `mc.json: crashes = 4`). Grader chạy lại với MSSV của bạn; số không khớp `evidence/` thì mất điểm phần D (RUBRIC), và câu dựa trên số sai khó được điểm đầy đủ. Không cần văn hay: một bảng nhỏ và hai, ba câu giải thích là đủ.

Thị trấn của bạn (`meta.json: _run.town`): **C** (`space.py`: đường ướt 20 %, người đi bộ ngó qua xe buýt 80 %, xe chuyển làn phanh sau khi vào 15 %, khe chấp nhận trung vị 10 m: chuyển làn bạo hơn thị trấn A).

Mọi số dưới đây lấy từ một lần `python run_all.py all` đầy đủ (không `--quick`). Số nào lấy từ lần chạy thêm ngoài `evidence/` được ghi rõ **[chạy thêm]**.

## 1. Phát lại log có trung thực không? (bước A)

| `replay.json` | Giá trị |
|---|---|
| `n` / `n_used` | 400 / 399 |
| `dropped_human_crash_logs` (log người lái tự va chạm, bỏ) | 1 |
| `replay.crashes` (planner trên log phát lại) | **110**: `rear` 97, `side` 13, `front` 0 |
| `reactive.crashes` (planner trong thế giới phản ứng) | **0** |
| `crash_in_both` | 0 |

88 % va chạm của "phát lại" là `rear`, và cả 110 đều là **va chạm giả**: không vụ nào xảy ra khi thế giới phản ứng (`crash_in_both = 0`). Planner giữ khoảng cách rộng hơn người lái (`T` 1,6 s so với 1,1 s của người) nên phanh sớm và mạnh hơn. Trong log, xe bám đuôi TG chạy đúng quỹ đạo đã ghi khi bám theo **người lái**: nó không biết xe ta đã phanh sớm hơn, nên không phanh theo và tông vào đuôi. Ngoài đời TG sẽ phản ứng, nên đây là lỗi của phương pháp kiểm thử, không phải của planner. 13 vụ `side` cũng cùng cơ chế: xe khác chuyển làn vào đúng chỗ mà trong log còn trống, vì xe ta trong log đã ở chỗ khác.

Ảnh `evidence/shots/replay_rear.png` (`viewer.json` R-01: TG tông đuôi xe ta khi phát lại log). Ví dụ `replay_only[0]` (kịch bản 0, `contact = rear`): C2 chuyển làn vào trước xe ta lúc t ≈ 3,9 s (`cutT0` 3,87, `cutDs` +7,9 m). TG bám rất sát (`tgT` 0,60 s). Planner phanh để mở lại khe 1,6 s, còn TG trong log vẫn chạy như khi người lái chỉ giảm tốc nhẹ, nên đâm vào đuôi. Khi thế giới phản ứng, TG phanh theo, va chạm biến mất, và TTC nhỏ nhất của xe ta vẫn còn 2,30 s (`min_ttc_reactive` = 2,30). `world.json` cho thấy cùng hiện tượng trên câu chuyện mặc định: "phát lại" có va chạm `('ego', 'TG', 14.8)`, "phản ứng" có 0.

## 2. Kịch bản bất khả thi (bước B)

| Luật trong `is_feasible` | Căn cứ vật lý |
|---|---|
| Gia tốc ngang của chuyển làn `(π/cutDur)²·3,5/2 > μg` thì loại | Chuyển làn 3,5 m theo cung cosin; lực ngang tối đa của lốp là μ·m·g. Vượt quá thì xe trượt, không ai lái ra được quỹ đạo đó. |
| `cutBrake > μg` thì loại | Gia tốc phanh không vượt được μg (ma sát lốp: 7,8 m/s² khô, 4,9 m/s² ướt). |
| `pedV > 2,5 m/s` thì loại | Người **đi bộ** nhanh nhất khoảng 2 m/s; trên 2,5 m/s là chạy, nằm ngoài mô hình người qua đường của lab. |
| `cutV ∉ (0, 22]` thì loại | Xe chuyển làn phải đang chạy, và không vượt ~80 km/h trên phố đô thị này. |

`hunt.json: infeasible`: `crosswalk_block` **15**, `rear_ended` **15**, `bus_bypass` 1, `cut_in` 0 (`mc.json: infeasible` = 8/3 000 dưới phân phối tự nhiên). CEM bị kéo về phía đó vì kịch bản bất khả thi thường là kịch bản "nguy hiểm" nhất theo đúng điểm của cuộc săn. Với `crosswalk_block`, người đi càng nhanh thì càng xuất hiện bất ngờ và khoảng phanh còn lại (`ped_room`) càng nhỏ, nên elite dồn về `pedV` cao và đi qua ngưỡng 2,5 m/s. Với `rear_ended`, điểm là TTC của TG lên xe ta, tăng khi xe ta phải phanh gấp, tức khi C2 chen vào cực nhanh (`cutDur` ngắn, gia tốc ngang > μg) hoặc phanh mạnh hơn μg ngay sau khi vào làn.

Nếu **đếm** chúng là va chạm: (a) p bị thổi phồng bằng những vụ không thể xảy ra; (b) CEM hội tụ hẳn vào vùng bất khả thi, nên đề xuất q mất tác dụng cho vùng thật; (c) bước E sẽ chỉnh planner để chống "ma", có thể đổi lấy rủi ro thật ở chỗ khác.

## 3. Monte Carlo cho bạn biết gì (bước C1)

`mc.json`: **k = 3**, **n = 3 000**, **p̂ = 1,00·10⁻³**, Wilson 95 % **[3,40·10⁻⁴, 2,94·10⁻³]**; `modes`: `bus_bypass` 2, `crosswalk_block` 1.

Cần bao nhiêu lần chạy để khoảng hẹp còn ±20 % của p: 1,96·√(p/n) = 0,2·p ⇒ n = (1,96/0,2)²/p ≈ **96/p**. Với p̂ = 10⁻³ thì n ≈ **96 000 lần chạy**, gấp 32 lần ngân sách MC hiện có.

Các kiểu **chưa hề xuất hiện** trong 3 000 lần: `cut_in`, `rear_ended` và `other` (k = 0). Con số đúng cần báo **không phải 0** mà là cận trên Wilson: `mc.json: per_mode.cut_in.hi` = `per_mode.rear_ended.hi` = `per_mode.other.hi` = **1,28·10⁻³** (tương đương quy tắc 3/n ≈ 10⁻³). Ta chỉ biết mỗi kiểu này < 1,3·10⁻³ với độ tin cậy 95 %.

## 4. Importance sampling, ESS và độ suy biến (bước C3)

`estimate.json`, n = 600 mỗi kiểu mỗi seed:

| Kiểu | Seed | `p` | `hits` | `ess` | `top_share` |
|---|---|---|---|---|---|
| bus_bypass | 0 | 1,13·10⁻³ | 49 | 127 | 0,27 |
| bus_bypass | 1 | 9,92·10⁻⁴ | 53 | 121 | 0,17 |
| bus_bypass | 2 | 9,21·10⁻⁴ | 56 | 128 | 0,12 |
| crosswalk_block | 0 | 1,42·10⁻⁶ | 105 | 102 | 0,37 |
| crosswalk_block | 1 | 5,23·10⁻⁶ | 108 | 147 | 0,22 |
| crosswalk_block | 2 | 1,75·10⁻⁶ | 113 | 112 | 0,20 |
| cut_in | 0 / 1 / 2 | 0 → `upper_zero_hits` **0,025** | 0 | 128 / 136 / 140 | 0 |
| rear_ended | 0 / 1 / 2 | 0 → `upper_zero_hits` **0,025** | 0 | 132 / 130 / 133 | 0 |
| **total** | 0 / 1 / 2 | 1,13·10⁻³ / 9,97·10⁻⁴ / 9,22·10⁻⁴ | | | |
| `other` (từ `mc.json`) | | k = 0/3 000, `hi` = 1,28·10⁻³ | | | |

**Ba seed có đồng ý không?** Với `bus_bypass` thì có: chênh 1,23 lần, và khớp MC (`mc.json`: 2/3 000 = 6,7·10⁻⁴, Wilson [1,8·10⁻⁴, 2,4·10⁻³] trong `per_mode.bus_bypass`). Với `crosswalk_block` thì **không**: các seed chênh 3,7 lần, và cả ba **thấp hơn MC hàng trăm lần**. MC thấy 1 vụ/3 000 (Wilson [5,9·10⁻⁵, 1,9·10⁻³]); cận dưới đó còn lớn hơn cận trên IS (9,0·10⁻⁶ ở seed 1). Đây là bẫy 3: cuộc săn tìm ra **một** cách chặn lối dễ tìm nhưng cực hiếm, còn q bỏ sót vùng mà MC tình cờ gặp. Ở đó không mẫu nào nặng ký, nên `top_share` vẫn thấp (≤ 0,37) mà ước lượng vẫn sai. `top_share` thấp không chứng minh ước lượng đúng.

**`top_share` > 0,5:** trong `estimate.json` không seed nào vượt 0,5 (cao nhất 0,37: `crosswalk_block`, seed 0). Hiện tượng này lộ rõ trong `fix.json`: `crosswalk_block` có `top_share` 0,89 (`baseline_train`), 0,96 (`baseline_test`) và 0,64 (`chosen_test`); `bus_bypass` của `baseline_test` là 0,54. Khi `top_share` = 0,96, p = mean(w·y) gần như chỉ là w của **một** mẫu chia cho n. Con số khi đó phụ thuộc vào việc mẫu đó có được rút trúng hay không, và ±1,96·se hẹp sai vì se cũng chỉ do mẫu đó quyết định.

**Ngân sách:** `estimate.json: budget` = **16 800** lần chạy (4 cuộc săn × 8 vòng × 300, cộng 3 seed × 4 kiểu × 600). Để MC đạt cùng sai số tương đối như IS seed 0, cần n ≈ (1−p)/(p·(se/p)²):
- `bus_bypass` (se/p = 0,33): ≈ **8 000** lần chạy MC, tức IS không lợi hơn MC nhiều với một kiểu cỡ 10⁻³;
- `crosswalk_block` (p = 1,42·10⁻⁶, se/p = 0,42): ≈ **4·10⁶** lần chạy MC. Lợi thế của IS nằm ở kiểu rất hiếm, nhưng chỉ khi q phủ đúng vùng va chạm (xem đoạn trên).

**`other`** lấy từ Monte Carlo C1 (`estimate.json: other.source = "mc.json"`, k = 0, `hi` = 1,28·10⁻³). Ta không săn nó vì `other` là phần còn lại không đồng nhất (va chạm với W1a, E5…), nên không có một điểm nguy hiểm liên tục nào để CEM leo theo. Một cuộc săn chung sẽ lại co về `bus_bypass`.

## 5. Các kiểu thất bại (bước C2 + 3D)

| Cuộc săn (`hunt.json`) | `found` | `infeasible` | `stopped_tightening` |
|---|---|---|---|
| bus_bypass | bus_bypass 223 | 1 | true |
| cut_in | **bus_bypass 28** (0 cut_in) | 0 | false |
| crosswalk_block | crosswalk_block 77, bus_bypass 2 | 15 | true |
| rear_ended | **crosswalk_block 9, bus_bypass 5** (0 rear_ended) | 15 | false |

**`bus_bypass`** (ảnh `evidence/shots/bus_bypass.png`, `viewer.json` B-01, t ≈ 12,4 s): xe buýt đỗ ở làn E2 trước vạch. C2 đi sau xe buýt, chậm lại rồi lách sang trái, lấn vào làn xe ta để vượt. Xe ta tới gần vạch bị che với khoảng 7,8 m/s (≈ `cautionV` 7,5). Nó chỉ coi C2 là "xe phía trước" khi C2 đã lấn sâu hơn `cutinLat` = 2,15 m, nên phanh muộn. Va chạm `front` ở x ≈ 90 m: xe ta còn 5,6 m/s, C2 chỉ còn 1,6 m/s.

**`crosswalk_block`** (ảnh `evidence/shots/crosswalk_block.png`, C-01, t ≈ 21,45 s): đường ướt (μ = 0,5). Người đi bộ P1 bước ra ở vạch x ≈ 122 m. Xe ta phanh và dừng hẳn ở x ≈ 119,7 m, nhưng xe dài 4,6 m nên đầu xe nằm trên lối qua đường. P1 không bước vào thân xe, nên đứng lại giữa làn (z ≈ 3,5 m). Không có tiếp xúc, nhưng planner phanh quá muộn và chặn lối người đi bộ.

**Kiểu không tìm thấy:** `cut_in` và `rear_ended`. Cả hai cuộc săn đều trôi sang kiểu khác và không hội tụ (`stopped_tightening = false`). Mức tin: chỉ biết chúng **hiếm hơn** `bus_bypass` dưới policy gốc, với cận trên MC 1,28·10⁻³ (`mc.json`) và cận IS 0,025 mỗi seed (`upper_zero_hits`). Đó **không** phải bằng chứng là chúng không xảy ra: điểm nguy hiểm (TTC) của chúng có thể không dẫn được CEM tới vùng va chạm thật.

**Một cuộc săn chung cho mọi kiểu** sẽ co về kiểu dễ nhất. Bằng chứng: cuộc săn `cut_in` tìm ra 28 `bus_bypass` và không vụ `cut_in` nào; cuộc săn `bus_bypass` cho 223 va chạm cùng một kiểu. Một cuộc săn chung sẽ báo gần như toàn `bus_bypass`, và `crosswalk_block` (cần đường ướt, người bước ra đúng lúc) sẽ bị bỏ sót. Vì thế mỗi kiểu cần một cuộc săn với điểm nguy hiểm riêng.

## 6. Sửa planner (bước E + 3D)

Policy được chọn bằng `choose_policy` trên TRAIN: **`{"cutinLat": 3.3}`** (`fix.json: chosen.policy`). Mình không thêm ứng viên nào ngoài 12 ứng viên có sẵn.

| `fix.json` | `fail` | `fail_hi_seen` | `fail_hi` (hợp lệ) | `unseen` | `progress` [m] | `harsh` [khung/24 s] |
|---|---|---|---|---|---|---|
| Gốc, TRAIN (`baseline_train`) | 1,59·10⁻³ | 3,10·10⁻³ | 3,10·10⁻³ | — | 215,3 | 8,23 |
| cutinLat 3,3, TRAIN (`chosen`) | 1,72·10⁻⁵ | 4,70·10⁻⁵ | 4,70·10⁻⁵ | bus_bypass | 214,8 | 7,81 |
| **cutinLat 3,3, TEST** (`chosen_test`) | **4,81·10⁻⁷** | **1,13·10⁻⁶** | **3,88·10⁻²** | bus_bypass | 210,0 | 8,12 |
| **Gốc, TEST** (`baseline_test`) | **1,40·10⁻³** | **3,04·10⁻³** | **4,32·10⁻³** | — | 210,1 | 8,98 |

TRAIN chỉ dùng để sàng; số cần báo là số TEST: seed mới, một cuộc săn mới nhắm vào chính policy được chọn, và va chạm ngoài kiểu được đếm trên 3 000 lần chạy tự nhiên (`other_crashes.n`).

Trên TEST, bản sửa xoá được `bus_bypass` khỏi mẫu (0 hit) mà gần như không mất quãng đường (210,0 so với 210,1 m) và phanh gấp ít hơn (8,12 so với 8,98). Nhưng **`fail_hi` hợp lệ của bản sửa (3,88·10⁻²) lại lớn hơn cả `fail` (1,40·10⁻³) và `fail_hi` (4,32·10⁻³) của policy gốc**. Lý do: kiểu `unseen` (`bus_bypass`) phải cộng cận `upper_zero_hits` = 3/(0,2·400) = 0,0375. Không thấy va chạm trong 400 mẫu chỉ chứng minh được p < 3,75·10⁻², nên ngân sách này không chứng nhận được một kiểu đã biến mất dưới 10⁻². `fail_hi_seen` (1,13·10⁻⁶) là cái dữ liệu cho thấy, còn `fail_hi` là điều ta được phép khẳng định. Bản sửa có thể tốt hơn thật mà vẫn không chứng minh được bằng 400 mẫu; muốn vậy cần khoảng 15 000 lần chạy trên đúng vùng đó.

Ảnh có **bóng mờ**: `evidence/shots/fix_shadow.png` (kịch bản B-01, t = 12,4 s; xe đặc là policy đã chọn, bóng mờ là policy gốc. Vì các run B-xx trong `viewer.json` được ghim vào policy gốc, mình chụp từ một bản sao của `viewer.json` chỉ bỏ trường `policy` của B-01, để trình xem chạy policy đã chọn và lấy policy gốc làm bóng mờ). Xe ta với `cutinLat` 3,3 nhận ra C2 đang lấn sang từ khi C2 còn cách 3,3 m theo phương ngang, nên phanh sớm và dừng kịp. Bóng mờ (policy gốc, 2,15 m) phản ứng muộn và đâm vào C2.

**Núm "nghe thì an toàn hơn" nhưng tệ hơn:** `T` = 2,0 (giữ khoảng cách xa hơn) cho TRAIN `fail` = **2,22·10⁻³**, lớn hơn policy gốc 1,59·10⁻³. Tăng là do `bus_bypass`: p = 2,20·10⁻³ với 45 hit, so với 1,58·10⁻³ với 37 hit của policy gốc (`fix.json: table[8]`). Cơ chế, bằng phản ứng của thế giới: C2 chuyển làn khi thấy khe ≥ `cutGap`, và thị trấn C có trung vị chỉ 10 m. Xe ta để khoảng trống trước mặt rộng hơn thì C2 và các xe lách xe buýt thấy khe đủ rộng thường hơn, nên chen vào làn xe ta nhiều hơn, ngay sát đầu xe. Tương tự, giảm `cautionV` xuống 6,0 hay 6,5 gần như không giảm `bus_bypass` (1,11·10⁻³, 1,17·10⁻³) mà tăng phanh gấp từ 8,2 lên 10,4–12,0. Xe chậm lại trước xe buýt cũng là mở ra khe cho C2 lách vào.

## 7. Từ mô phỏng ra đường thật

| Giả định (`space.py`) | Dữ liệu thật thay nó, lấy từ đâu | Nếu sai thì con số sai theo hướng nào |
|---|---|---|
| **Khe chấp nhận của xe chuyển làn** `cutGap` (log-normal, trung vị 10 m ở thị trấn C, σ = 0,5) và `cutDur` (trung vị 4 s) | Quỹ đạo chuyển làn trong bộ dữ liệu drone/quỹ đạo: highD/inD/rounD (Đức), NGSIM US-101/I-80, Waymo Open Motion; trích khe phía trước/sau tại lúc bắt đầu chuyển làn và thời lượng chuyển làn. Tốt nhất là quay camera trên đúng loại phố đô thị Việt Nam có xe buýt đỗ. | `bus_bypass` là ~99 % tổng p. Nếu người thật chấp nhận khe **nhỏ hơn** (giao thông VN chen sát hơn) thì p bị **đánh giá thấp**; nếu khe lớn hơn thì bị đánh giá cao. Câu 6 cho thấy p nhạy với khe này. |
| **Tỉ lệ người đi bộ ngó qua đầu xe buýt** (`pedLook` = 0,80) và khe thời gian họ chấp nhận `pedGap` (trung vị 4,5 s) | Quan sát video tại các trạm xe buýt gần vạch qua đường: đếm tỉ lệ người bước ra mà không nhìn, đo khe họ chấp nhận (nghiên cứu gap acceptance của người đi bộ, dữ liệu inD có người đi bộ). | Nếu ít người ngó hơn 80 % thì `crosswalk_block` bị **đánh giá thấp**: nhiều người bước ra từ sau xe buýt mà xe ta không thấy. |
| **Đường ướt** (20 %, μ = 0,5 so với 0,8) | Số ngày mưa và giờ mưa theo trạm khí tượng (TP.HCM/Hà Nội mùa mưa có tỉ lệ cao hơn nhiều so với 20 %), kết hợp đo hệ số ma sát mặt đường (thiết bị đo kiểu GripTester/SCRIM) trên đúng loại nhựa đường. | `crosswalk_block` ở C-01 cần μ = 0,5. Nếu tỉ lệ ướt cao hơn hoặc μ ướt thấp hơn 0,5 thì `crosswalk_block` bị **đánh giá thấp**, và một số kịch bản đang bị `is_feasible` loại (gia tốc > μg) lại trở thành khả thi nếu μ cao hơn. |

## 8. Giới hạn

1. **Một con phố, một cấu hình:** chỉ có một xe buýt đỗ ở x ≈ 106 m ngay trước vạch qua đường x = 122 m, một xe chuyển làn C2, một người đi bộ P1 có kịch bản. Xác suất báo cáo là "mỗi lần đi qua đúng đoạn này", không phải mỗi km hay mỗi chuyến đi thật (ngã tư, xe máy, đỗ xe hai bên… không có).
2. **Người lái là IDM với gap acceptance đơn giản:** không có xe máy lách, không có người lái mất tập trung hay vượt đèn, không có tương tác nhiều bên. Xe khác luôn "lịch sự" (`civil`) và không bao giờ cố ý đâm. Cả `rear_ended` lẫn `cut_in` có thể hiếm chỉ vì hành vi người được mô hình hoá quá ngoan.
3. **Cảm biến hoàn hảo trừ phần xe buýt che:** không có nhiễu, độ trễ nhận thức hay phát hiện sai. Chỉ xe buýt chặn tầm nhìn, xe con thì không. Ngoài ra, chỉ phanh được thử như cách tránh (không đánh lái), và phân phối tự nhiên là giả định dạy học chứ không fit từ dữ liệu.

Nếu có thêm một tuần: (a) chạy MC lớn (~10⁵ lần, n ≈ 96/p) để kiểm tra trực tiếp chênh lệch MC–IS của `crosswalk_block`; (b) săn lại `crosswalk_block` với nhiều điểm khởi đầu và điểm nguy hiểm khác (khoảng cách đầu xe tới mép vạch) để tìm vùng va chạm mà q đã bỏ sót; (c) kiểm tra bản sửa `cutinLat` 3,3 với ~15 000 lần chạy trên vùng `bus_bypass` để thật sự chứng nhận p < 10⁻³; (d) thêm độ trễ nhận thức 0,3–0,5 s vào planner rồi đo lại.

## 9. Rắc rối của bạn (bước G + chương 10)

- **Rắc rối:** một **người băng ngang giữa khối nhà, không có vạch** (`hazard.json`: `type: ped`, `side: S`). Người này bước ra từ lề phía S trên đoạn x 150–300 m, sau xe buýt đỗ (x ≈ 106) và vạch qua đường thật (x = 122), khi xe ta còn cách `trig` mét. Ngoài đời đây là người qua đường tắt sang quán bên kia đường, đặc biệt phổ biến ở phố thương mại Việt Nam, và thường đi nhanh, chấp nhận khe ngắn.
- **Căn cứ:** planner thận trọng **ở đúng chỗ đã biết**. `cautionV`/`cautionRange` gắn với vạch bị xe buýt che, và `pedBuffer` dùng để dự đoán người đi bộ trên vạch. Trên đoạn đường thẳng không có vạch, xe ta chạy ~7–10 m/s và không có logic dự đoán một người đang tiến về làn của mình. Mình thử 10 biến thể (xe đạp lạng, xe chết máy, phanh gấp, người băng ngang hai phía). Chỉ biến thể này còn va chạm `avoidable` dưới cả policy đã sửa; xe đạp lạng từ làn E2 thì được `cutinLat` 3,3 cứu hết **[chạy thêm, seed khác, ngoài `evidence/`]**.
- **Giả định** (`hazard.json: why`):
  - `h0_v` 1,0–2,0 m/s: phân vị 15 tốc độ đi bộ 0,97 m/s (≥ 65 tuổi) và 1,25 m/s (14–64 tuổi) (Knoblauch et al. 1996, TRR 1538). Người băng ngang giữa khối hay đi nhanh nên cận trên lấy 2,0, dưới ngưỡng "chạy" 2,5 m/s của `is_feasible`.
  - `h0_trig` 10–40 m: ở ~12 m/s là khe ~1–3 s, phần đuôi nguy hiểm của người chấp nhận khe ngắn.
  - `h0_x` 150–300 m: đoạn phố thẳng sau vạch thật, không có vạch.
  - `exposure` = 0,02/chuyến: nghiên cứu Milwaukee (SSTI 2026) cho ~1/6 lượt sang đường là giữa khối. **Giả định** ~30 lượt sang đường/giờ trên đoạn phố này ⇒ ~5 lượt giữa khối/giờ; xe ta qua đoạn 150 m trong ~15 s ⇒ 5·15/3600 ≈ 0,02. Mắt xích yếu nhất là con số 30 lượt/giờ: cần đếm thật trên đoạn phố đó để thay.
- **Kết quả** (`mine.json`; `accepted` = **0,995**, 1 492 khả thi / 1 500, `infeasible` = 8; `exact.planner` = `[ego, H0, 19.8]`):

| `mine.json` | Policy gốc (`baseline`) | cutinLat 3,3 (`chosen`) |
|---|---|---|
| `crashes` | 10 | 10 |
| `avoidable` (lỗi planner) | **9** | **10** |
| `unavoidable` | 0 | 0 |
| `unattributed` | 1 | 0 |
| `low_speed` | 0 | 0 |
| `avoidable_foreknown` | 9 | 10 |
| P(lỗi planner \| rắc rối) = `p` [Wilson `lo`, `hi`] | 6,03·10⁻³ [3,18·10⁻³, 1,14·10⁻²] | 6,70·10⁻³ [3,64·10⁻³, 1,23·10⁻²] |
| Mỗi chuyến `per_trip` (cận `per_trip_hi`) | 1,21·10⁻⁴ (2,28·10⁻⁴) | 1,34·10⁻⁴ (2,46·10⁻⁴) |

  So với lách xe buýt (`estimate.json`, `bus_bypass` ≈ 0,92–1,13·10⁻³ mỗi lần chạy, tức mỗi chuyến qua đoạn phố này), rắc rối của mình hiếm hơn khoảng **8 lần mỗi chuyến** dưới policy gốc. Nhưng sau bản sửa, `bus_bypass` không còn thấy trong mẫu (câu 6) còn rắc rối này **không giảm**, nên nó có thể trở thành nguồn lỗi planner lớn nhất còn lại. Con số mỗi chuyến tỉ lệ thuận với `exposure`, tức với giả định 30 lượt/giờ.

  Va chạm `avoidable` điển hình (`mine.json: chosen.examples`): người đi bộ bước ra khi xe ta còn ~36–39 m (`h0_trig`), đi 1,8–2,0 m/s, ở x ≈ 160 m. Xe ta va chạm ~3,5 s sau, vẫn còn 7–9 m/s, trong khi phanh hết cỡ ngay khi thấy (`brake_t` = `fire_t`) thì dừng kịp. `avoidable_foreknown` = `avoidable`: người không bị che, nên vấn đề không phải tầm nhìn mà là planner **không phản ứng**.

  **"Tránh được" là chuẩn lạc quan theo hai hướng:** (1) người phanh "biết trước": phanh tối đa ngay từ khung hình đầu tiên rắc rối bắt đầu và lọt vào cảm biến, không có thời gian nhận thức hay quyết định. Một planner thật cần vài trăm ms để nhận ra người đó sẽ băng qua, nên `avoidable` **đếm nhiều hơn** số vụ một planner thật tránh được (đó là cận trên của "lỗi planner"). (2) Phép thử chỉ có **phanh**, không thử đánh lái tránh. Vài vụ đang bị xếp là không tránh được có thể tránh được bằng một cú lách nhẹ, nên theo hướng này nó **đếm ít đi**. Ở rắc rối này `unavoidable` = 0, nên chiều (1) là chiều đáng lo.
- **Hình:** `evidence/shots/hazard_ped.png` (chương 9, `viewer.json` kịch bản GB-01, một va chạm `avoidable` từ `mine.json: baseline.examples`, đúng lúc xe ta chạm người băng ngang, t = 20,2 s; người đi bộ đã đi hết làn ngoài và đang ở trước đầu xe, trong làn xe ta).
- **Bạn tự lái thì sao:** chưa thử.
- **Cách sửa:** không núm nào trong 9 núm sửa được **[chạy thêm trên đúng 1 492 phiên bản của `mine`, ngoài `evidence/`]**. `pedBuffer` = 6,0 (mức tối đa) vẫn để 9 vụ `avoidable`; `pedBuffer` chỉ dùng cho người trên vạch. `v0` = 10 m/s chỉ giảm còn 8 vụ, nhưng va chạm `unattributed` tăng từ 1 lên 10 (xe chạy chậm bị xe khác đâm), và mất quãng đường. Planner cần thêm **logic mới**: theo dõi mọi người đi bộ trên lòng đường hoặc ở mép đường (không chỉ trên vạch), dự đoán thời điểm họ vào làn xe ta từ vận tốc và hướng đi. Nếu thời điểm đó rơi vào khoảng xe ta đi tới (cộng biên an toàn), thì giảm tốc hoặc nhường ngay, như cách nó đã làm với P1 ở vạch qua đường.
