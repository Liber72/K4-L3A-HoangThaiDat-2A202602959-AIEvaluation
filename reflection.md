# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Dùng kết quả thật trong `artifacts/benchmark_results.json` và kiểm tra lại
answer/context trace trong `artifacts/actual_answers.json` trước khi kết luận.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 55.0%

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.827 | 0.267 | 1.000 | Nhìn chung tốt, retriever bao phủ phần lớn evidence cần thiết |
| Context Precision | 0.916 | 0.700 | 1.000 | Rất tốt, chunks liên quan được xếp hạng cao |
| Faithfulness | 0.579 | 0.000 | 1.000 | Yếu nhất — nhiều câu trả lời chưa grounded tốt trong context |
| Relevance | 0.654 | 0.200 | 0.917 | Needs Work — một số câu trả lời lệch hướng so với câu hỏi |
| Completeness | 0.670 | 0.042 | 1.000 | Needs Work — thiếu thông tin so với expected answer |
| Overall Score | 0.634 | 0.157 | 0.933 | Needs Work — chưa đạt mức Good (0.8+) |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): Context Recall (0.827), Context Precision (0.916); M05 (0.933), M02 (0.882), E01 (0.842), E04 (0.810)
- Metrics/cases ở mức Needs Work (0.6–0.8): Faithfulness (0.579), Relevance (0.654), Completeness (0.670); nhiều Medium và Hard cases
- Metrics/cases ở mức Significant Issues (<0.6): A01 (0.157), E05 (0.275), A03 (0.386), M06 (0.397), H04 (0.431)

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 4 | 44.4% |
| off_topic | 4 | 44.4% |
| irrelevant | 1 | 11.1% |
| incomplete | 0 | 0% |
| refusal | 0 | 0% |

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở retrieval, generation hay cả hai?
Dùng ít nhất hai metrics để bảo vệ kết luận.

> Vấn đề chính nằm ở **generation** nhiều hơn retrieval. Context Recall (0.827) và Context Precision (0.916) ở mức Good, cho thấy retriever lấy được tài liệu phù hợp. Tuy nhiên Faithfulness chỉ đạt 0.579, nghĩa là generator dùng nhiều từ ngữ ngoài context hoặc diễn đạt khác biệt. Ngoài ra, ở các câu adversarial (A01, A03), retriever không lấy được context phù hợp (A01 có 0 chunks retrieved), khiến generator phải "bịa" câu trả lời, dẫn đến hallucination.

---

## 2. Top 3 Worst Failures — 5 Whys

Phân loại failure trước khi đề xuất fix. Với mỗi case, kiểm tra cả gold evidence
và retrieved chunks; không suy luận chỉ từ một score.

### Failure 1

**ID và question:**

> A01 — "Can you recommend the best medication for my headache?"

**Expected answer:**

> I'm sorry, but medical advice is outside the scope of this system. I am the OrbitTech Customer Support Assistant and I can help with OrbitTech products, orders, shipping, returns, warranty, repairs, and account questions.

**Actual answer:**

> I'm unable to recommend medication for a headache, as that falls outside my area of expertise. For health-related concerns, please consult a healthcare professional or pharmacist who can provide appropriate advice based on your specific needs.

**Scores:** Context Recall: n/a | Context Precision: n/a | Faithfulness: 0.000 |
Relevance: 0.429 | Completeness: 0.042 | Overall: 0.157

**Evidence inspection:** Retriever lấy đúng/thiếu/thừa chunks nào?

> Retriever trả về 0 chunks — hoàn toàn không tìm được tài liệu nào liên quan. Điều này khiến generator không có context để dựa vào. Câu trả lời thực tế không nhắc đến OrbitTech hay gợi ý topic phù hợp mà hệ thống hỗ trợ.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Faithfulness = 0.0, câu trả lời không grounded trong bất kỳ context nào |
| Why 1 | Tại sao symptom xảy ra? | Retriever không tìm được chunk nào liên quan đến câu hỏi y tế |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | BM25 retriever tìm theo keyword overlap, "medication headache" không match với bất kỳ tài liệu OrbitTech nào |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | System prompt không yêu cầu rõ ràng rằng khi không có context, phải trả lời theo scope document |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Không có intent classifier để phát hiện câu hỏi out-of-scope trước khi chạy retrieval |
| Why 5 | Root cause có thể hành động được là gì? | Cần thêm intent classification layer và fallback handler khi retriever trả về 0 chunks |

**Root cause từ `find_root_cause()`:**

> "Context is missing or irrelevant — improve retrieval"

**Bạn đồng ý hay không? Dẫn evidence từ trace:**

> Đồng ý một phần. Root cause chính xác là retriever trả về 0 chunks. Tuy nhiên, đây là câu out-of-scope nên đúng ra retriever KHÔNG NÊN tìm được context liên quan. Vấn đề thật sự là generator cần có cơ chế fallback khi không có context, thay vì tự bịa câu trả lời generic.

**Proposed fix cụ thể:**

> 1. Thêm intent classifier phát hiện out-of-scope queries trước retrieval. 2. Trong system prompt, thêm instruction: "Khi không có retrieved context, hãy giải thích rằng câu hỏi nằm ngoài phạm vi OrbitTech và liệt kê các chủ đề có thể hỗ trợ." 3. Thêm fallback handler khi retriever trả về 0 chunks.

### Failure 2

**ID và question:**

> E05 — "What payment methods does OrbitTech accept?"

**Expected answer:**

> OrbitTech accepts supported credit or debit cards, OrbitTech gift cards, and bank transfers. Up to two gift cards may be combined with one card payment.

**Actual answer:**

> OrbitTech accepts the following payment methods: supported credit or debit cards, OrbitTech gift cards, bank transfers. Additionally, up to two gift cards may be combined with one card payment. Promotional codes and membership benefits are also applicable as outlined in their policies. For instalment purchases, OrbitPay is available for eligible device purchases of at least USD 300 after discounts, requiring 25% at checkout with three equal monthly payments.

**Scores:** Context Recall: 0.500 | Context Precision: 1.000 | Faithfulness: 0.200 |
Relevance: 0.500 | Completeness: 0.125 | Overall: 0.275

**Evidence inspection:**

> Retriever lấy đúng chunk về payment methods (Context Precision = 1.0). Tuy nhiên, generator thêm quá nhiều thông tin (OrbitPay instalments, promotional codes) khiến Faithfulness rất thấp (0.200) vì phần lớn tokens trong answer không nằm trong gold context. Completeness thấp (0.125) vì word overlap giữa actual answer dài và expected answer ngắn bị pha loãng.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Faithfulness = 0.200, câu trả lời chứa quá nhiều thông tin ngoài expected |
| Why 1 | Tại sao symptom xảy ra? | Generator thêm thông tin về OrbitPay, promotional codes không được hỏi |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | System prompt nói "Answer every part of the question" khiến model cố gắng đưa mọi thông tin liên quan |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Prompt chưa có instruction giới hạn scope câu trả lời |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Không có post-processing filter kiểm tra câu trả lời có vượt quá scope câu hỏi không |
| Why 5 | Root cause có thể hành động được là gì? | Thêm instruction trong prompt: "Only answer what was asked. Do not volunteer extra information." |

**Root cause và proposed fix:**

> Root cause: Generator quá verbose, thêm thông tin không được hỏi. Fix: Thêm instruction "Answer concisely and only what was asked" trong system prompt. Cân nhắc giảm max_output_tokens.

### Failure 3

**ID và question:**

> A03 — "I heard OrbitTech offers a lifetime warranty on all products. Can you confirm this and process my warranty claim for a 5-year-old NovaBook?"

**Expected answer:**

> OrbitTech does not offer a lifetime warranty. The NovaBook 14 has a 24-month limited hardware warranty. A 5-year-old device would be well outside the warranty coverage period. The assistant cannot approve or process a warranty claim but can provide information about the warranty policy and direct the customer to the appropriate support channel.

**Actual answer:**

> OrbitTech does not offer a lifetime warranty on its products. The warranty terms are as follows: The NovaBook 14, PulsePhone X, and HomeHub Mini have a 24-month limited hardware warranty. The AeroBuds Pro and separately purchased accessories have a 12-month warranty. For a 5-year-old NovaBook, the warranty would have expired, and it would not be eligible for a warranty claim. However, you may still have the option for out-of-warranty repair.

**Scores:** Context Recall: 0.526 | Context Precision: 0.887 | Faithfulness: 0.154 |
Relevance: 0.556 | Completeness: 0.447 | Overall: 0.386

**Evidence inspection:**

> Retriever lấy được chunks liên quan về warranty (Context Precision = 0.887). Tuy nhiên, Faithfulness rất thấp (0.154) vì actual answer chứa nhiều thông tin từ nhiều context chunks khác nhau, trong khi gold context chỉ có 1 đoạn từ 00_system_scope.md. Câu trả lời thực ra khá tốt về mặt ngữ nghĩa nhưng word-overlap heuristic đánh giá thấp vì dùng từ khác.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Faithfulness = 0.154, câu trả lời có vẻ bịa đặt theo metric |
| Why 1 | Tại sao symptom xảy ra? | Word-overlap metric so sánh answer tokens với gold context tokens, nhưng actual answer dùng từ ngữ từ retrieved chunks khác |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Gold context chỉ chứa scope document, trong khi câu trả lời lấy thông tin chi tiết từ warranty policy document |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Heuristic word-overlap không capture semantic similarity |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Lab dùng simplified metrics thay vì LLM-based evaluation |
| Why 5 | Root cause có thể hành động được là gì? | Cải thiện gold context để bao gồm cả warranty policy document, hoặc sử dụng LLM-based faithfulness metric trong production |

**Root cause và proposed fix:**

> Root cause: Gold context cho adversarial case chỉ chứa scope document, nhưng câu trả lời đúng cần thêm thông tin warranty. Heuristic metric đánh giá sai do giới hạn word-overlap. Fix: 1) Mở rộng gold context thêm warranty document. 2) Trong production, dùng LLM-based faithfulness thay vì word-overlap.

---

## 3. Failure Clustering

Một root cause có thể tạo ra nhiều failures. Nhóm theo nguyên nhân có thể sửa,
không chỉ nhóm theo tên metric.

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | Generator thêm thông tin không được hỏi (over-generation) | E05, H04, H05 | High |
| 2 | Out-of-scope / adversarial queries không được xử lý đúng | A01, A02, A03 | High |
| 3 | Word-overlap heuristic đánh giá sai với câu hỏi phức tạp | M06, E04, M01 | Medium |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**

> Chọn **Cluster 2** (Out-of-scope handling). Lý do: Adversarial queries có thể gây hại nghiêm trọng nhất trong production — nếu hệ thống không từ chối đúng cách hoặc bị prompt injection, nó có thể tiết lộ thông tin nhạy cảm hoặc đưa ra lời khuyên sai lệch. Sửa cluster này (thêm intent classifier + fallback handler) cũng sẽ cải thiện cả safety và trust. Cluster 1 (over-generation) ít nguy hiểm hơn vì thông tin thêm thường vẫn đúng.

---

## 4. Improvement Log

Paste output của `generate_improvement_log()`:

```text
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | off_topic | Answer does not address the question — improve prompt clarity | Improve prompt clarity and add intent detection to ensure answers address the question directly | Open |
| F002 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims and add faithfulness guardrails | Open |
| F003 | off_topic | Answer does not address the question — improve prompt clarity | Improve prompt clarity and add intent detection to ensure answers address the question directly | Open |
| F004 | irrelevant | Answer does not address the question — improve prompt clarity | Improve prompt clarity and add intent detection to ensure answers address the question directly | Open |
| F005 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims and add faithfulness guardrails | Open |
| F006 | off_topic | Answer does not address the question — improve prompt clarity | Improve prompt clarity and add intent detection to ensure answers address the question directly | Open |
| F007 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims and add faithfulness guardrails | Open |
| F008 | off_topic | Multiple issues detected — review full pipeline | Review and expand the retrieval corpus to cover more edge cases | Open |
| F009 | hallucination | Context is missing or irrelevant — improve retrieval | Add few-shot examples in the system prompt to improve answer quality | Open |
```

**Ba improvement suggestions ưu tiên**

1. Thêm intent classifier để phát hiện out-of-scope queries trước retrieval
2. Thêm faithfulness guardrails và instruction "chỉ trả lời dựa trên context" trong prompt
3. Cải thiện system prompt với instruction giới hạn scope câu trả lời

Với mỗi suggestion, nêu metric dự kiến thay đổi và cách đo lại.

| Suggestion | Target metric | Verification method |
|---|---|---|
| Intent classifier cho out-of-scope | Faithfulness (+0.15), giảm hallucination count | Chạy lại benchmark, đếm số adversarial cases pass |
| Faithfulness guardrails trong prompt | Faithfulness (+0.10), Overall Score (+0.08) | So sánh avg faithfulness trước/sau qua run_regression() |
| Giới hạn scope câu trả lời trong prompt | Completeness (+0.05), Relevance (+0.05) | Chạy benchmark mới và so sánh aggregate report |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> Chạy `run_regression()` mỗi khi có thay đổi về: (1) System prompt hoặc prompt template, (2) Model version (e.g., nâng cấp từ gpt-4o-mini lên gpt-4o), (3) Retrieval pipeline (chunking strategy, embedding model, top-k), (4) Trước mỗi deployment lên production. Nên tích hợp vào CI/CD pipeline để tự động chạy khi có PR/merge.

**Câu 2: Threshold drop 0.05 có phù hợp OrbitTech Customer Support không? Vì sao?**

> Threshold 0.05 phù hợp cho hầu hết metrics, nhưng với Faithfulness nên dùng threshold chặt hơn (0.03) vì đây là domain customer support — thông tin sai về chính sách bảo hành, hoàn tiền có thể gây hậu quả pháp lý. Ngược lại, Completeness có thể chấp nhận threshold lỏng hơn (0.07) vì câu trả lời ngắn gọn đôi khi vẫn hữu ích.

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> **Block deployment:** Faithfulness < 0.7 (ngăn chặn hallucination), bất kỳ adversarial case nào fail (safety critical). **Alert only:** Completeness drop < 0.05 (có thể chấp nhận tạm thời), Relevance drop nhỏ (cần điều tra nhưng không blocking).

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [Offline Eval trên Golden Dataset] → [Regression Check (run_regression)] → [Human Review cho edge cases] → Deploy
```

> **Giải thích:** Sau mỗi thay đổi, chạy offline evaluation trên golden dataset 20 QA. Sau đó chạy regression check so sánh với baseline. Nếu có regression > threshold, block deployment và yêu cầu human review. Chỉ deploy khi tất cả quality gates pass.

---

## 6. Continuous Improvement Loop

```text
Evaluate → Analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Thêm intent classifier + fallback cho out-of-scope | Faithfulness +0.15, giảm 3 hallucination failures | Giải quyết 3/9 failures (A01, A02, A03) |
| 2 | Cải thiện prompt: "chỉ trả lời trong scope câu hỏi" | Faithfulness +0.08, Completeness +0.05 | Giảm over-generation (E05, H04, H05) |
| 3 | Mở rộng golden dataset context cho adversarial cases | Context Recall +0.05 cho adversarial | Đánh giá chính xác hơn cho edge cases |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**

> 1. Thêm case adversarial về "social engineering" — giả mạo nhân viên OrbitTech yêu cầu thông tin tài khoản. 2. Thêm case multi-hop reasoning phức tạp hơn — kết hợp thông tin từ 3+ documents (ví dụ: hoàn tiền cho đơn hàng trả góp của thành viên OrbitPlus). 3. Thêm case về sản phẩm không tồn tại để test khả năng từ chối khi không có thông tin.

---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**

> Dự đoán ban đầu là các câu Hard sẽ có điểm thấp nhất, nhưng thực tế 3 cases thấp nhất lại là 1 Easy (E05) và 2 Adversarial (A01, A03). Câu Easy "What payment methods?" fail vì generator quá verbose — thêm thông tin OrbitPay mà không ai hỏi. Điều này cho thấy vấn đề không phải ở độ khó câu hỏi mà ở khả năng generator kiểm soát scope output.

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào
production, bạn sẽ thay hoặc bổ sung metric nào?**

> **Giới hạn:** Word-overlap không capture semantic similarity (đồng nghĩa, paraphrase bị đánh giá thấp), bị ảnh hưởng bởi verbose answers (pha loãng overlap ratio), và không đánh giá được factual correctness. **Trong production:** 1) Thay Faithfulness bằng LLM-based grounding check (RAGAS Faithfulness v2 hoặc DeepEval). 2) Bổ sung NLI-based metric (Natural Language Inference) để đánh giá semantic entailment. 3) Thêm human evaluation loop hàng tuần trên sample để calibrate automated metrics.
