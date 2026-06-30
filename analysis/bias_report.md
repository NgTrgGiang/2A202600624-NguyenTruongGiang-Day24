# LLM Judge Bias Report — Phase B

**Sinh viên:** Nguyễn Trường Giang
**Ngày:** 2026-06-30
**Judge model:** gpt-4o-mini

---

## 1. Pairwise Judge Results

*(So sánh `model_answer` (A) vs `ground_truth` (B) trên 10 câu human-labeled. Bảng dưới trích 5 cặp.)*

| # | ID | Question (tóm tắt) | Winner | Reasoning tóm tắt |
|---|---|---|---|---|
| 1 | 1 | Nghỉ bao nhiêu ngày khi kết hôn? | B | B (đáp án chuẩn) đầy đủ hơn, nêu rõ "không trừ phép năm" |
| 2 | 5 | Mua thiết bị 55tr cần ai duyệt? | B | A sai (Director), B đúng (CEO vì >50tr) |
| 3 | 21 | Senior 9 năm: phép + lương? | B | Cả hai đúng nhưng B nêu rõ cách tính 15+3 và dải lương |
| 4 | 41 | Nghỉ bao nhiêu ngày phép năm? | B | A trả lời theo v2023 (sai), B đúng v2024 hiện hành |
| 5 | 50 | Dùng VPN cá nhân khi WFH? | B | A trả lời "được" (sai), B đúng: cấm, phải dùng WireGuard công ty |

→ Cả **10/10** cặp judge chọn **B (ground_truth)** — đáp án chuẩn luôn chính xác & đầy đủ hơn câu trả lời pipeline.

---

## 2. Swap-and-Average Results

*(Chạy `swap_and_average()` trên cùng 10 cặp — đổi thứ tự A/B để phát hiện position bias.)*

| # | ID | Pass 1 Winner | Pass 2 Winner (đã convert) | Final | Position Consistent? |
|---|---|---|---|---|---|
| 1 | 1 | B | B | B | ✅ Yes |
| 2 | 5 | B | B | B | ✅ Yes |
| 3 | 12 | B | B | B | ✅ Yes |
| 4 | 21 | B | B | B | ✅ Yes |
| 5 | 23 | B | B | B | ✅ Yes |
| 6 | 29 | B | B | B | ✅ Yes |
| 7 | 33 | B | B | B | ✅ Yes |
| 8 | 41 | B | B | B | ✅ Yes |
| 9 | 46 | B | B | B | ✅ Yes |
| 10 | 50 | B | B | B | ✅ Yes |

**Position bias rate:** **0.0%** (0/10 case không nhất quán) — judge cho kết quả y hệt khi đảo thứ tự A/B.

---

## 3. Cohen's κ Analysis

**Human labels:** `human_labels_10q.json` (10 câu: 6 label=1, 4 label=0)
**Judge labels:** chấm bằng `grade_answer_vs_reference()` — so `model_answer` với `ground_truth`, trả 1 (đúng) / 0 (sai).

| Question ID | Human Label | Judge Label | Agree? |
|---|---|---|---|
| 1 | 1 | 1 | ✅ |
| 5 | 0 | 0 | ✅ |
| 12 | 1 | 1 | ✅ |
| 21 | 1 | 1 | ✅ |
| 23 | 1 | 1 | ✅ |
| 29 | 0 | 0 | ✅ |
| 33 | 1 | 1 | ✅ |
| 41 | 0 | 0 | ✅ |
| 46 | 1 | 1 | ✅ |
| 50 | 0 | 0 | ✅ |

**Cohen's κ:** **1.000** (observed agreement 10/10)
**Interpretation:** **almost perfect** (Landis–Koch: >0.8). → ✅ **Bonus Phase B** (κ > 0.6).

---

## 4. Verbosity Bias

Trong các case có winner rõ ràng (không tie), tổng decisive = 10:
- A thắng + A dài hơn B: **0 / 10**
- B thắng + B dài hơn A: **10 / 10**
- **Verbosity bias rate:** **1.0 (100%)**

**Kết luận:** Con số 1.0 trông đáng lo, nhưng ở thí nghiệm này nó **bị nhiễu (confounded)**: B luôn là
`ground_truth` — vừa **đúng hơn** vừa **dài/đầy đủ hơn** model_answer. Vì vậy "B dài hơn + B thắng" xảy ra
100% là do B đúng hơn, **không tách bạch được** đó là do độ dài hay do chất lượng. Muốn đo verbosity bias
"sạch", cần so hai câu trả lời **cùng độ chính xác nhưng khác độ dài**. Dù vậy, đây vẫn là cảnh báo: LLM judge
pairwise có xu hướng ưu ái câu dài/đầy đủ — nguy hiểm khi một câu dài-nhưng-sai có thể thắng câu ngắn-nhưng-đúng.

---

## 5. Nhận xét chung

- **κ = 1.0 > 0.6** → trên 10 câu này, LLM judge (reference-based) **đồng thuận tuyệt đối** với người. Judge
  rất đáng tin khi được cung cấp đáp án chuẩn để đối chiếu.
- **Position bias = 0%** → swap-and-average xác nhận judge ổn định, không thiên vị theo vị trí A/B. Với corpus
  này position bias không phải vấn đề, nhưng swap-and-average vẫn nên giữ như một lớp bảo hiểm rẻ tiền.
- **Verbosity bias** là rủi ro thực: nên thêm tiêu chí "phạt câu trả lời thừa/lan man" vào prompt và ưu tiên
  tính đúng hơn độ dài.
- **Trong production:** (1) luôn chấm theo *reference* khi có ground truth thay vì pairwise thuần; (2) dùng
  swap-and-average cho mọi so sánh pairwise; (3) lấy mẫu định kỳ cho người review để re-validate κ; (4) không
  để một LLM vừa sinh vừa tự chấm chính nó (tránh self-preference bias).
