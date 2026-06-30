# Failure Cluster Analysis — Phase A

**Sinh viên:** Nguyễn Trường Giang
**Ngày:** 2026-06-30

---

## 1. Aggregate RAGAS Scores theo Distribution

| Metric | factual | multi_hop | adversarial |
|---|---|---|---|
| faithfulness | 0.908 | **0.432** | 0.800 |
| answer_relevancy | 0.793 | 0.712 | 0.702 |
| context_precision | 0.967 | 0.967 | 0.967 |
| context_recall | 0.950 | 0.796 | 0.717 |
| **avg_score** | **0.905** | **0.727** | **0.796** |

→ `factual` mạnh nhất (0.905), `multi_hop` yếu nhất (0.727) chủ yếu vì **faithfulness sụp xuống 0.432**.
`context_precision` cao đều (0.967) ở cả 3 — reranker lọc nhiễu tốt; điểm yếu nằm ở **sinh câu trả lời**, không phải retrieval.

---

## 2. Bottom 10 Questions

| Rank | ID | Distribution | Question (tóm tắt) | avg_score | worst_metric |
|---|---|---|---|---|---|
| 1 | 39 | multi_hop | So sánh yêu cầu mật khẩu v1.0 vs v2.0 (độ dài, hạn đổi, MFA) | 0.125 | faithfulness |
| 2 | 30 | multi_hop | So sánh quyền lợi bảo hiểm thử việc vs chính thức | 0.375 | faithfulness |
| 3 | 33 | multi_hop | Manager 12 năm: tổng phụ cấp + ngày phép v2024 | 0.375 | faithfulness |
| 4 | 50 | adversarial | Dùng VPN cá nhân (NordVPN) khi WFH? | 0.417 | faithfulness |
| 5 | 44 | adversarial | Bao lâu phải đổi mật khẩu một lần? | 0.458 | faithfulness |
| 6 | 9 | factual | Nam nghỉ bao nhiêu ngày khi vợ sinh con? | 0.500 | faithfulness |
| 7 | 31 | multi_hop | Công tác trong nước, khách sạn 1.5tr/đêm, công ty trả tối đa? | 0.622 | faithfulness |
| 8 | 24 | multi_hop | Tạm ứng 15tr, quá hạn 5 ngày, phạt bao nhiêu? | 0.698 | faithfulness |
| 9 | 38 | multi_hop | Công tác nước ngoài, khách sạn 200 USD/đêm × 3 | 0.712 | faithfulness |
| 10 | 28 | multi_hop | Lead: tổng phụ cấp ăn trưa + điện thoại | 0.716 | faithfulness |

→ **7/10 câu tệ nhất là multi_hop**, và **10/10 đều có worst_metric = faithfulness**: pipeline tự "bịa"
con số khi phải tính toán/so sánh nhiều bước (so sánh version, cộng phụ cấp, tính phí phạt pro-rata).

---

## 3. Failure Cluster Matrix

*(Mỗi ô = số câu có worst_metric = row, thuộc distribution = col)*

| worst_metric | factual | multi_hop | adversarial | Total |
|---|---|---|---|---|
| faithfulness | 3 | **15** | 2 | **20** |
| answer_relevancy | 13 | 1 | 1 | 15 |
| context_precision | 2 | 0 | 0 | 2 |
| context_recall | 2 | 4 | **7** | 13 |

---

## 4. Dominant Failure Analysis

**Dominant metric (theo số câu):** `faithfulness` (20/50 câu có đây là metric yếu nhất)
**Cụm failure nặng nhất:** `multi_hop × faithfulness` (15 câu)

> Báo cáo tự động ghi `dominant_failure_distribution = factual` chỉ vì hàm đếm tổng số worst-metric, mà
> factual có 13 câu rơi vào `answer_relevancy`. Nhưng đây là **artifact**: các câu factual vốn điểm rất cao
> (avg 0.905), `answer_relevancy` chỉ tình cờ là metric *thấp tương đối* trong 4 metric gần như hoàn hảo —
> không phải failure thực sự. Điểm yếu **thật sự** của pipeline là cụm **multi_hop × faithfulness**:
> 15/20 câu multi_hop có faithfulness là metric tệ nhất, kéo faithfulness trung bình của multi_hop xuống
> 0.432. Nguyên nhân: câu multi_hop buộc LLM tính toán/suy luận nhiều bước (so sánh v1.0–v2.0, cộng phụ cấp
> theo cấp bậc, tính phí phạt pro-rata) → LLM hay chèn con số không có trong context (hallucination).

---

## 5. Suggested Fixes

| Metric yếu | Root cause | Suggested fix |
|---|---|---|
| faithfulness | LLM hallucinating khi tính toán multi-hop | Siết system prompt ("chỉ dùng số có trong context, không tự suy diễn"), temperature=0, thêm bước chain-of-thought có trích dẫn nguồn từng con số |
| context_recall | Thiếu chunk liên quan (adversarial 0.717) | Tăng recall: thêm BM25 weight, nâng top_k trước rerank, chunk theo version để không bỏ sót tài liệu hiện hành |
| context_precision | Đã rất tốt (0.967) | Giữ nguyên reranker (FlashRank) |
| answer_relevancy | Câu trả lời lệch ý hỏi (chủ yếu ở factual, mức nhẹ) | Tinh chỉnh prompt template để bám sát đúng câu hỏi |

---

## 6. Nhận xét về Adversarial Distribution

- **avg_score: adversarial (0.796) < factual (0.905)** → đúng kỳ vọng (✅ **bonus Phase A**): pipeline làm
  tệ hơn rõ rệt trên các bẫy version-conflict/negation so với câu tra cứu thẳng.
- Đặc trưng của adversarial: **context_recall thấp nhất (0.717)** và 7/10 câu adversarial có worst_metric =
  context_recall → khi corpus có cả v2023 lẫn v2024 (hoặc v1.0/v2.0), pipeline **lấy nhầm tài liệu cũ** hoặc
  lấy thiếu bản hiện hành.
- Trong bottom-10 có **2 câu adversarial**: Q50 (VPN cá nhân — policy contradiction trap) và Q44 (đổi mật khẩu
  — version conflict v1.0/v2.0). Cả hai bị faithfulness kéo xuống vì pipeline trả lời theo policy sai phiên bản
  hoặc trả lời "có" cho câu vốn phải phủ định.
- **Khuyến nghị:** thêm metadata `version` + `effective_date` cho chunk và filter "chỉ tài liệu hiệu lực hiện
  hành" trước khi đưa vào LLM, để xử lý dứt điểm nhóm version-conflict.
