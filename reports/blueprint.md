# CI/CD Blueprint: RAG Eval + Guardrail Stack

**Sinh viên:** Lam Thanh Bao
**Ngày:** 2026-08-26

**Ghi chú hạ tầng:** Lab được cấu hình gốc dùng OpenAI `gpt-4o-mini` qua OpenRouter. Trong quá trình chạy, tài khoản OpenRouter gặp giới hạn credit (cả model trả phí lẫn model free — free tier giới hạn cứng 50 request/ngày nếu tài khoản chưa nạp credit). Đã chuyển toàn bộ 3 phase sang **DeepSeek** (`deepseek-chat`, API tương thích OpenAI, key có sẵn từ Day 18) để hoàn thành đủ 200+ LLM call cần thiết cho RAGAS 50q + Judge + Guardrails.

---

## Guard Stack Architecture

```
User Input
    │
    ▼ (~?ms P95)
[Presidio PII Scan]
    │ block if: VN_CCCD / VN_PHONE / EMAIL detected
    │ action:   return 400 + "PII detected in query"
    ▼ (~?ms P95)
[NeMo Input Rail]
    │ block if: off-topic / jailbreak / prompt injection
    │ action:   return 503 + refuse message
    ▼
[RAG Pipeline (Day 18)]
    │ M1 Chunk → M2 Search → M3 Rerank → GPT-4o-mini
    ▼
[NeMo Output Rail]
    │ flag if:  PII in response / sensitive content
    │ action:   replace with safe response
    ▼
User Response
```

---

## Latency Budget

*(Kết quả thực đo từ `measure_p95_latency()`, `reports/guard_results.json`, n=10 adversarial inputs)*

| Layer | P50 (ms) | P95 (ms) | P99 (ms) | Budget |
|---|---|---|---|---|
| Presidio PII | 15.8 | 17.8 | 17.8 | <10ms |
| NeMo Input Rail | 946.5 | 1302.7 | 1302.7 | <300ms |
| RAG Pipeline | *(không đo trong Task 12)* | — | — | <2000ms |
| NeMo Output Rail | *(không đo riêng — cùng cơ chế với Input Rail)* | — | — | <300ms |
| **Total Guard (Presidio+NeMo input)** | 962.1 | **1318.3** | 1318.3 | **<500ms** |

**Budget OK?** [x] No

**Comment:** NeMo Input Rail là bottleneck rõ rệt (P95=1303ms, chiếm ~99% tổng latency), vượt xa ngân sách 300ms và kéo tổng vượt 500ms gần 3 lần. Nguyên nhân: mỗi lần check phải gọi 1 lượt LLM thật (`generate_user_intent`) qua DeepSeek — độ trễ mạng quốc tế (server DeepSeek ở Trung Quốc, gọi từ VN) cộng với thời gian suy luận. Presidio (17.8ms) cũng cao hơn mức <10ms kỳ vọng vì dùng `en_core_web_lg` (spaCy full model) — có thể đổi sang `en_core_web_sm` để giảm latency. Cách tối ưu cho production: (1) cache kết quả classification cho các câu hỏi lặp lại/tương tự, (2) dùng model nhỏ hơn/nhanh hơn cho input rail (không cần model mạnh để phát hiện jailbreak), (3) chạy Presidio và NeMo song song (asyncio.gather) thay vì tuần tự, (4) cân nhắc self-hosted embedding-based classifier thay vì LLM call cho các rule đơn giản như off-topic detection.

---

## CI/CD Gates (phải pass trước khi merge to main)

```yaml
# .github/workflows/rag_eval.yml
- name: RAGAS Quality Gate
  run: python src/phase_a_ragas.py
  env:
    MIN_FAITHFULNESS: 0.75
    MIN_AVG_SCORE: 0.65

- name: Guardrail Gate
  run: pytest tests/test_phase_c.py -k "test_adversarial_suite_pass_rate"
  # phải ≥ 15/20 (75%)

- name: Latency Gate
  run: python -c "from src.phase_c_guard import measure_p95_latency; ..."
  # P95 total < 500ms
```

---

## Monitoring Dashboard (production)

| Metric | Alert Threshold | Action |
|---|---|---|
| RAGAS faithfulness (daily sample) | < 0.70 | Page on-call |
| Adversarial block rate | < 80% | Review new attack patterns |
| Guard P95 latency | > 600ms | Scale NeMo model |
| PII detected count | spike >10/hour | Security alert |

---

## Kết quả thực tế từ Lab

| | Kết quả |
|---|---|
| RAGAS avg_score (50q) | factual 0.881 / multi_hop 0.630 / adversarial 0.691 |
| Worst metric | faithfulness (16/50 câu, tập trung ở multi_hop: 11/20) |
| Dominant failure distribution | factual và multi_hop hòa (20/20 câu mỗi bên), nguyên nhân khác nhau — xem `analysis/failure_clusters.md` |
| Cohen's κ | 0.444 (moderate, chưa đạt bonus >0.6) |
| Adversarial pass rate | 20 / 20 (100%, đạt bonus ≥18/20) |
| Guard P95 latency | 1318.3 ms (vượt ngân sách 500ms — bottleneck: NeMo Input Rail 1302.7ms) |

**Bonus đạt được:** Phase A (adversarial avg 0.691 < factual avg 0.881 ✓, +4) và Phase C (pass rate 100% ≥ 90% ✓, +3). Phase B chưa đạt bonus (κ=0.444 < 0.6).

---

## Nhận xét & Cải tiến

> Stack hoạt động tốt ở phần phát hiện PII (Presidio với custom VN_CCCD/VN_PHONE recognizer đạt độ chính xác cao sau khi giới hạn entity types để tránh false positive từ recognizer tiếng Anh mặc định) và guardrail chặn adversarial input (100% pass rate). Điểm cần cải thiện rõ nhất là RAGAS faithfulness ở câu multi-hop — pipeline Day 18 cần cải thiện khả năng tổng hợp nhiều tài liệu cùng lúc thay vì suy diễn khi thiếu ngữ cảnh, và LLM judge (κ=0.444) chưa đủ tin cậy để thay thế con người hoàn toàn, nên dùng làm bộ lọc sơ bộ. Nếu deploy production thực sự, tôi sẽ: (1) thêm cơ chế cache cho NeMo input rail để giảm P95 latency xuống dưới 300ms, (2) tách riêng một model nhỏ/nhanh chỉ để classify jailbreak/off-topic thay vì dùng chung model chính, (3) thêm bước "self-check facts" sau RAG để giảm hallucination ở câu multi-hop, và (4) log toàn bộ case position-bias/tie từ LLM judge để con người review định kỳ thay vì tin tưởng tuyệt đối vào judge tự động.
