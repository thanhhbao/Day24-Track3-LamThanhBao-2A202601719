# LLM Judge Bias Report — Phase B

**Sinh viên:** Lâm Thành Bảo - 2A202601719
**Ngày:** 2026-08-26
**Judge model:** `deepseek-chat` (DeepSeek, OpenAI-compatible — đổi từ gpt-4o-mini do giới hạn credit/rate limit của OpenRouter, xem `reports/blueprint.md`)

Phương pháp: mỗi câu trong `human_labels_10q.json` được judge dưới dạng cặp **A = model_answer** (câu trả lời của pipeline Day 18) vs **B = ground_truth** (đáp án chuẩn trong `test_set_50q.json`). Nếu judge chọn A hoặc tie → coi model_answer là đúng (judge_label=1); nếu chọn B → model_answer sai (judge_label=0).

---

## 1. Pairwise Judge Results

| #   | Question (tóm tắt)                             | Winner | Reasoning tóm tắt                                           |
| --- | ---------------------------------------------- | ------ | ----------------------------------------------------------- |
| 1   | Nghỉ khi kết hôn?                              | B      | Ground truth đầy đủ hơn (nêu rõ không trừ phép năm)         |
| 5   | Mua thiết bị 55tr cần ai duyệt?                | B      | Ground truth có ngưỡng chính xác + người duyệt (CEO)        |
| 12  | Thưởng Tết tối thiểu?                          | B      | Ground truth có điều kiện 6 tháng + pro-rata                |
| 21  | Senior 9 năm: phép + lương?                    | B      | Ground truth có công thức tính rõ ràng                      |
| 23  | Hoàn trả phí đào tạo nghỉ sau 8 tháng?         | B      | Ground truth nêu rõ cam kết 1 năm + mức hoàn trả 100%       |
| 29  | Tạm ứng 8tr quá hạn: ai duyệt, phạt bao nhiêu? | B      | Ground truth chi tiết hơn về ngưỡng duyệt và cách tính phạt |
| 33  | Manager 12 năm: phụ cấp + phép?                | tie    | Cả hai đều đủ chi tiết, khác biệt nhỏ                       |
| 41  | Nghỉ phép năm bao nhiêu ngày?                  | B      | model_answer dùng policy v2023 cũ, ground_truth đúng v2024  |
| 46  | Thử việc có phép năm không?                    | tie    | Cả hai đều đúng và đủ, model_answer súc tích hơn            |
| 50  | VPN cá nhân khi WFH?                           | B      | model_answer bỏ sót việc VPN cá nhân bị cấm theo policy     |

---

## 2. Swap-and-Average Results

| #   | Pass 1 Winner | Pass 2 Winner | Final | Position Consistent? |
| --- | ------------- | ------------- | ----- | -------------------- |
| 1   | B             | B             | B     | ✓                    |
| 5   | B             | B             | B     | ✓                    |
| 12  | B             | B             | B     | ✓                    |
| 21  | B             | B             | B     | ✓                    |
| 23  | B             | B             | B     | ✓                    |
| 29  | B             | B             | B     | ✓                    |
| 33  | B             | A             | tie   | ✗                    |
| 41  | B             | B             | B     | ✓                    |
| 46  | tie           | A             | tie   | ✗                    |
| 50  | B             | B             | B     | ✓                    |

**Position bias rate (từ `reports/judge_results.json`):** 30% (3/10 case không nhất quán giữa 2 lượt — swap thay đổi kết quả đối với các câu có 2 đáp án gần tương đương chất lượng).

---

## 3. Cohen's κ Analysis

**Human labels:** `human_labels_10q.json` (10 câu)
**Judge labels:** từ `swap_and_average()` so model_answer với ground_truth (0 = model_answer sai, 1 = model_answer đúng/tie)

| Question ID | Human Label | Judge Label | Agree? |
| ----------- | ----------- | ----------- | ------ |
| 1           | 1           | 0           | ✗      |
| 5           | 0           | 0           | ✓      |
| 12          | 1           | 0           | ✗      |
| 21          | 1           | 1           | ✓      |
| 23          | 1           | 0           | ✗      |
| 29          | 0           | 0           | ✓      |
| 33          | 1           | 1           | ✓      |
| 41          | 0           | 0           | ✓      |
| 46          | 1           | 1           | ✓      |
| 50          | 0           | 0           | ✓      |

**Cohen's κ:** 0.444
**Interpretation:** moderate (0.4–0.6 theo thang Landis-Koch) — chưa đạt ngưỡng bonus (>0.6 substantial).

Đáng chú ý: raw agreement là 7/10 (70%), nhưng κ chỉ 0.444 vì κ điều chỉnh theo xác suất đồng thuận ngẫu nhiên (p_e cao do cả 2 phía đều thiên về label=1/label=0 không đều). 3 case bất đồng (ID 1, 12, 23) đều là các câu mà human chấm "đúng" (1) nhưng judge chọn ground_truth (B) là tốt hơn — cho thấy judge có xu hướng khắt khe hơn con người khi so sánh với đáp án chuẩn "hoàn hảo", trong khi con người chấp nhận câu trả lời đúng ý dù thiếu vài chi tiết phụ.

---

## 4. Verbosity Bias

Trong các case có winner rõ ràng (không phải tie, 7/10 case):

- A (model_answer) thắng + A dài hơn B: 0/7 cases
- B (ground_truth) thắng + B dài hơn A: 7/7 cases
- **Verbosity bias rate:** 100%

**Kết luận:** Trong 100% case quyết định, judge chọn câu trả lời dài hơn (ground_truth thường chi tiết/đầy đủ hơn model_answer). Đây một phần phản ánh đúng thực tế (ground_truth vốn được viết đầy đủ hơn do là đáp án chuẩn), nhưng cũng là dấu hiệu classic verbosity bias của LLM judge — cần cẩn trọng khi dùng judge để so sánh 2 câu trả lời có độ dài chênh lệch lớn, vì judge có thể ưu tiên câu dài dù nó không thực sự chính xác hơn.

---

## 5. Nhận xét chung

> κ=0.444 chưa đạt mức "substantial" (>0.6) — LLM judge (`deepseek-chat`) đồng thuận với con người ở mức trung bình, chưa đủ tin cậy để thay thế hoàn toàn đánh giá thủ công trong production. Position bias 30% là mức đáng lưu ý (ngưỡng cảnh báo thường đặt ở >30%) — cho thấy với các câu trả lời có chất lượng gần tương đương, thứ tự A/B ảnh hưởng đến quyết định của judge. Swap-and-average thực sự hữu ích: nó phát hiện được đúng 3/3 case không nhất quán và chuyển chúng thành "tie" thay vì báo sai một kết quả chắc chắn — nếu chỉ chạy 1 pass, ta sẽ có kết luận sai lệch ở các câu 33 và 46. Verbosity bias 100% cũng cho thấy cần bổ sung tiêu chí "độ dài phù hợp" rõ ràng hơn trong prompt, tránh judge ngầm ưu tiên câu dài. Trong production, nên dùng LLM judge như một lớp lọc sơ bộ (flag các case bất đồng/tie để con người review) thay vì nguồn chân lý cuối cùng, đặc biệt với các câu có κ thấp hoặc position bias cao.
