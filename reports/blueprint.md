# CI/CD Blueprint: RAG Eval + Guardrail Stack

**Sinh viên:** Nguyễn Trường Giang
**Ngày:** 2026-06-30

---

## Guard Stack Architecture

```
User Input
    │
    ▼ (~11ms P95)
[Presidio PII Scan]      ← allowlist VN_CCCD / VN_PHONE / EMAIL_ADDRESS, score ≥ 0.5
    │ block if: PII thật được phát hiện
    │ action:   return 400 + "PII detected in query"
    ▼ (~1410ms P95)
[NeMo Input Rail]        ← LLM self-check (gpt-4o-mini), không dùng embeddings/annoy
    │ block if: jailbreak / off-topic / prompt injection / yêu cầu PII người khác
    │ action:   return 503 + refuse message
    ▼
[RAG Pipeline (Day 18)]
    │ M1 Chunk → M2 Hybrid Search (BM25+bge-m3) → M3 Rerank (FlashRank) → GPT-4o-mini
    ▼
[NeMo Output Rail]       ← self-check output: chặn nếu lộ PII/dữ liệu nhạy cảm
    │ flag if:  PII / sensitive content trong response
    │ action:   replace with safe response
    ▼
User Response
```

---

## Latency Budget

*(Đo thực tế bằng `measure_p95_latency()` — Task 12, n_runs=10)*

| Layer | P50 (ms) | P95 (ms) | P99 (ms) | Budget | Đạt? |
|---|---|---|---|---|---|
| Presidio PII | 8.49 | 11.30 | 11.30 | <10ms | ~đạt (sát ngưỡng) |
| NeMo Input Rail | 717.21 | 1410.28 | 1410.28 | <300ms | ❌ vượt |
| RAG Pipeline | — | ~12.000* | — | <2000ms | (không đo trong Task 12) |
| NeMo Output Rail | ~717 | ~1410 | — | <300ms | ❌ vượt |
| **Total Guard (Presidio+NeMo input)** | 725.32 | **1418.94** | 1418.94 | **<500ms** | ❌ vượt |

*\* RAG pipeline không nằm trong Task 12 (chỉ đo guard layers). Con số ~12s/query là trung bình quan sát từ
`setup_answers.py` (50 query / 597s), đã bao gồm cả LLM generation.*

**Budget OK?** [ ] Yes / [x] No
**Comment:** Bottleneck là **NeMo self-check** (~1410ms P95) — mỗi rail là một lời gọi LLM gpt-4o-mini. Presidio
gần như miễn phí (~11ms, local regex). Cách tối ưu: (1) chạy Presidio và NeMo **song song** rồi hợp kết quả;
(2) **cache** quyết định self-check theo hash của input; (3) thay self-check LLM bằng **classifier nhẹ**
(embedding + logistic, hoặc Llama-Guard on-prem) cho phần lớn input, chỉ escalate sang LLM khi mơ hồ;
(4) gộp input+output check thành 1 lời gọi. Với các biện pháp này, P95 guard có thể kéo về < 500ms.

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

**Kết quả gate trên kết quả lab này:**
- [ ] ❌ RAGAS faithfulness ≥ 0.75 → **0.70** (đạt ở factual 0.91 nhưng bị multi_hop 0.43 kéo xuống) → **FAIL**
- [x] ✅ Adversarial pass rate ≥ 90% (18/20) → **20/20 = 100%** → **PASS**
- [x] ✅ avg_score ≥ 0.65 → **0.81** → **PASS**
- [ ] ❌ P95 total guard latency < 500ms → **1419ms** → **FAIL**

→ Nếu áp chuẩn production, PR này **chưa merge được**: phải xử lý hallucination ở multi_hop (faithfulness) và
giảm latency NeMo trước. Đây là đúng giá trị của eval gate — chặn regression trước khi lên prod.

---

## Monitoring Dashboard (production)

| Metric | Alert Threshold | Action |
|---|---|---|
| RAGAS faithfulness (daily sample) | < 0.70 | Page on-call |
| Adversarial block rate | < 80% | Review new attack patterns |
| Guard P95 latency | > 600ms | Scale / cache NeMo, song song hóa |
| PII detected count | spike >10/hour | Security alert |

---

## Kết quả thực tế từ Lab

| | Kết quả |
|---|---|
| RAGAS avg_score (50q) | **0.812** |
| Worst metric | **faithfulness** (overall 0.70; multi_hop chỉ 0.43) |
| Dominant failure distribution | **multi_hop** (cụm multi_hop × faithfulness, 15 câu) |
| Cohen's κ | **1.000** (almost perfect, ✅ > 0.6) |
| Adversarial pass rate | **20 / 20** (100%, ✅ ≥ 18/20) |
| Guard P95 latency | **1418.94 ms** (Presidio 11ms + NeMo 1410ms) |

---

## Nhận xét & Cải tiến

> **Hoạt động tốt:** retrieval rất chắc (context_precision 0.967 đều khắp), guardrail chặn 20/20 adversarial
> (Presidio bắt 4 PII thật, NeMo self-check bắt 16 jailbreak/off-topic/injection), và LLM-as-judge đồng thuận
> tuyệt đối với human (κ=1.0). **Cần cải thiện:** faithfulness ở multi_hop (0.43) — pipeline hallucinate con số
> khi phải tính toán nhiều bước; và latency NeMo (~1.4s) do mỗi rail là một lời gọi LLM.
>
> **Nếu deploy production thật**, tôi sẽ thay đổi: (1) thêm metadata `version/effective_date` cho chunk + filter
> "chỉ tài liệu hiệu lực" để dứt điểm nhóm version-conflict (v2023/v2024, v1.0/v2.0); (2) ép LLM trích dẫn nguồn
> cho từng con số (giảm hallucination, nâng faithfulness); (3) thay self-check LLM bằng classifier nhẹ + cache +
> song song hóa Presidio/NeMo để P95 guard < 500ms; (4) luôn chấm theo reference + swap-and-average, và lấy mẫu
> cho người review định kỳ để giữ κ cao theo thời gian.
