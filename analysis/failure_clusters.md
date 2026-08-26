# Failure Cluster Analysis — Phase A

**Sinh viên:** Lam Thanh Bao
**Ngày:** 2026-08-26

---

## 1. Aggregate RAGAS Scores theo Distribution

| Metric | factual | multi_hop | adversarial |
|---|---|---|---|
| faithfulness | 0.950 | 0.433 | 0.617 |
| answer_relevancy | 0.839 | 0.784 | 0.713 |
| context_precision | 0.883 | 0.600 | 0.817 |
| context_recall | 0.850 | 0.704 | 0.617 |
| **avg_score** | **0.881** | **0.630** | **0.691** |

(Judge LLM: `deepseek-chat`, embeddings: `BAAI/bge-m3` local, chạy trên 50/50 câu, 0 lỗi/NaN.)

---

## 2. Bottom 10 Questions

| Rank | Distribution | Question | avg_score | worst_metric |
|---|---|---|---|---|
| 1 | multi_hop | So sánh quyền lợi bảo hiểm giữa nhân viên thử việc và nhân viên chính thức. | 0.125 | faithfulness |
| 2 | multi_hop | Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào? | 0.208 | faithfulness |
| 3 | factual | Muốn mua thiết bị trị giá 55 triệu cần ai phê duyệt? | 0.250 | answer_relevancy |
| 4 | multi_hop | Nhân viên tự ý xóa malware khỏi máy và chia sẻ thông tin sự cố này trên Slack nội bộ. Vi phạm những chính sách nào cụ thể? | 0.265 | context_precision |
| 5 | adversarial | Nhân viên Manager có thể dùng VPN cá nhân (như NordVPN) khi WFH để tăng bảo mật thêm không? | 0.375 | faithfulness |
| 6 | adversarial | Nhân viên thử việc có được nghỉ phép năm không? | 0.417 | faithfulness |
| 7 | multi_hop | Nhân viên Manager có thâm niên 12 năm: tổng phụ cấp hàng tháng và số ngày phép năm theo v2024 là bao nhiêu? | 0.456 | context_precision |
| 8 | multi_hop | Nhân viên tạm ứng 8 triệu, chưa thanh toán sau 30 ngày (quá hạn 15 ngày). Ai phê duyệt khoản này và phí phạt là bao nhiêu? | 0.460 | context_precision |
| 9 | multi_hop | So sánh yêu cầu mật khẩu giữa policy v1.0 và v2.0 về độ dài tối thiểu, thời hạn đổi và MFA. | 0.465 | context_precision |
| 10 | factual | Nam nhân viên được nghỉ bao nhiêu ngày khi vợ sinh con? | 0.500 | faithfulness |

---

## 3. Failure Cluster Matrix

*(Mỗi ô = số câu có worst_metric = row, thuộc distribution = col)*

| worst_metric | factual | multi_hop | adversarial | Total |
|---|---|---|---|---|
| faithfulness | 1 | 11 | 4 | 16 |
| answer_relevancy | 10 | 1 | 0 | 11 |
| context_precision | 5 | 7 | 2 | 14 |
| context_recall | 4 | 1 | 4 | 9 |
| **Total** | **20** | **20** | **10** | **50** |

---

## 4. Dominant Failure Analysis

**Dominant distribution:** factual (thuật toán chọn `max()` đầu tiên khi hòa — factual và multi_hop cùng có 20/20 câu, nhưng phân bố nguyên nhân khác hẳn nhau, xem bên dưới)
**Dominant metric:** faithfulness (16/50 câu có faithfulness là điểm yếu nhất, chủ yếu tập trung ở multi_hop)

**Lý do phân tích:**

> `factual` và `multi_hop` hòa nhau về tổng số câu "tệ nhất" (20/20), nhưng vì lý do khác nhau: ở `factual`, 10/20 câu yếu nhất là do `answer_relevancy` — model trả lời đúng nội dung nhưng dài dòng/lạc đề một phần so với câu hỏi (ví dụ câu #3 "Muốn mua thiết bị trị giá 55 triệu" — RAG trả lời đúng ngưỡng phê duyệt nhưng không bám sát format câu hỏi). Ở `multi_hop`, 11/20 câu yếu nhất là do `faithfulness` — pipeline phải tổng hợp từ nhiều tài liệu (ví dụ tính ngày phép + lương theo thâm niên, so sánh bảo hiểm thử việc vs chính thức) và dễ bịa số liệu khi ngữ cảnh không đủ. Corpus HR tiếng Việt có nhiều version chồng chéo (nghỉ phép v2023/v2024, mật khẩu v1.0/v2.0) khiến câu multi-hop cần kết hợp đúng 2-3 chunk cùng lúc — nếu retrieval chỉ lấy 1 phần, LLM tự suy diễn phần còn thiếu → faithfulness thấp.

---

## 5. Suggested Fixes

| Metric yếu | Root cause | Suggested fix |
|---|---|---|
| faithfulness | LLM hallucinating khi ghép nhiều tài liệu (multi_hop) | Tighten system prompt yêu cầu trích dẫn nguồn, hạ temperature, thêm bước "self-check facts" sau khi sinh câu trả lời |
| context_recall | Missing relevant chunks (đặc biệt ở adversarial — version conflict) | Tăng RERANK_TOP_K, thêm BM25 song song với dense search cho câu có số/ngày tháng cụ thể |
| context_precision | Too many irrelevant chunks khi câu hỏi multi_hop cần nhiều field khác nhau (phép + lương + phụ cấp) | Thêm metadata filter theo policy version mới nhất, rerank ưu tiên chunk có nhiều field trùng khớp câu hỏi |
| answer_relevancy | Answer đúng nội dung nhưng lệch format câu hỏi (factual) | Cải thiện prompt template yêu cầu trả lời đúng trọng tâm câu hỏi, tránh thêm thông tin thừa |

---

## 6. Nhận xét về Adversarial Distribution

> `adversarial` đạt avg_score 0.691 — thấp hơn `factual` (0.881) nhưng cao hơn `multi_hop` (0.630). Điều này đạt đúng kỳ vọng của bộ test set: adversarial được thiết kế để "bẫy" pipeline bằng version conflict và negation trap, nên thấp hơn factual rõ rệt — pipeline đôi khi trả lời theo policy cũ (v2023) thay vì v2024 hiện hành, hoặc bị nhầm bởi câu phủ định. Trong bottom 10, có 2 câu adversarial: #5 "VPN cá nhân NordVPN khi WFH" (avg=0.375, faithfulness thấp nhất — model có thể đã trả lời sai là "được phép" trong khi chính sách VPN v1.3 cấm VPN cá nhân) và #6 "Nhân viên thử việc có được nghỉ phép năm không?" (avg=0.417 — câu phủ định/ranh giới dễ gây nhầm lẫn giữa "không được nghỉ phép năm" và "có thể xin nghỉ không lương"). Việc adversarial thấp hơn factual xác nhận pipeline Day 18 vẫn còn điểm yếu thực sự với version conflict, đúng như mục tiêu thiết kế của bộ test set này.
