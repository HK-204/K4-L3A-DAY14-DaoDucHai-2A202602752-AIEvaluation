# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 14:15–17:00

**Domain:** OrbitTech Store Customer Support

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 14:15–14:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (14:30–14:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Câu đàm thoại xã giao (greeting) hoặc câu hỏi out-of-scope cần model từ chối lịch sự bằng system instructions thay vì trích dẫn context. | Câu hỏi về chính sách cốt lõi (giá, hoàn tiền, bảo hành) nhưng câu trả lời bịa đặt thông tin không có trong tài liệu (hallucination). | Bổ sung hallucination guardrail; sửa system prompt nhấn mạnh chỉ trả lời dựa trên context; hạ temperature; kiểm tra groundedness. |
| Answer Relevance | Người dùng đưa ra câu hỏi bẫy tiền đề sai (false premise trap) hoặc mơ hồ, trợ lý tập trung đính chính/làm rõ thay vì trả lời theo tiền đề sai. | Người dùng hỏi một vấn đề cụ thể (như thời hạn trả hàng) nhưng câu trả lời lại nói sang phương thức thanh toán hoặc giới thiệu sản phẩm. | Cải thiện prompt phân tích intent câu hỏi (intent classification); bổ sung few-shot examples hướng dẫn trả lời trúng trọng tâm. |
| Context Recall | Câu hỏi tra cứu đơn giản (factual lookup) mà 1 chunk duy nhất đã đủ thông tin trả lời, không cần các chunk khác phải chứa toàn bộ ground-truth. | Câu hỏi đa bước (multi-step/multi-doc) đòi hỏi tổng hợp từ nhiều chính sách nhưng retriever bỏ sót tài liệu chứa thông tin trọng yếu. | Tăng top-k retrieval; chuyển từ thuần từ khóa (BM25) sang hybrid search (BM25 + Dense Embeddings); điều chỉnh kích thước và độ overlap của chunk. |
| Context Precision | Retriever lấy top-k lớn (ví dụ k=10) để tối ưu recall, chấp nhận có chunk phụ trợ xếp lẫn nhưng mô hình sinh vẫn lọc được thông tin. | Chunks liên quan trực tiếp bị xếp ở vị trí cuối hoặc bị chôn vùi dưới nhiều chunk nhiễu, khiến LLM bị hiện tượng "lost in the middle" hoặc trả lời sai. | Thêm bước Reranking (Cross-Encoder reranker hoặc query-overlap rerank) để đẩy chunk quan trọng lên đầu; đặt threshold lọc similarity score. |
| Completeness | Người dùng chỉ hỏi một câu ngắn gọn, trợ lý trả lời đúng ý chính mà không liệt kê danh sách dài các trường hợp ngoại lệ không được hỏi. | Câu hỏi yêu cầu đầy đủ quy trình hoặc điều kiện bồi hoàn nhưng trợ lý bỏ sót các điều kiện bắt buộc, gây hiểu lầm nghiêm trọng cho khách hàng. | Bổ sung hướng dẫn định dạng có cấu trúc (bullet points/checklist); tăng max_tokens khi trả lời; kiểm tra độ bao phủ thông tin trước khi phản hồi. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*
> - **Thiết kế thực nghiệm (Pairwise Comparison with Order Swap):**
>   - Chuẩn bị một tập 20–30 cặp câu trả lời $(A, B)$ cho cùng một danh sách câu hỏi hỗ trợ khách hàng.
>   - **Condition 1 (Order AB):** Đưa vào prompt cho LLM Judge: "Candidate 1: Answer A, Candidate 2: Answer B. Hãy đánh giá câu trả lời nào tốt hơn hoặc cho điểm từng câu."
>   - **Condition 2 (Order BA):** Đảo ngược hoàn toàn thứ tự đưa vào: "Candidate 1: Answer B, Candidate 2: Answer A. Hãy đánh giá câu trả lời nào tốt hơn hoặc cho điểm từng câu."
> - **Đo lường & Đánh giá:**
>   - Thống kê tỷ lệ Candidate 1 được chọn hoặc có điểm cao hơn ở cả 2 condition: $P(\text{Win} \mid \text{Position 1})$.
>   - Nếu tỷ lệ Candidate 1 luôn thắng áp đảo (ví dụ $> 60–65\%$) bất kể nội dung là A hay B, hệ thống mắc phải Position Bias rõ rệt.
>   - **Khắc phục:** Áp dụng position-swapping (chấm cả 2 chiều và lấy trung bình hoặc chỉ chấp nhận thắng khi thắng ở cả 2 vị trí).

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
> 1. **Bổ sung tiêu chí Conciseness & Information Density:** Quy định rõ trong rubric rằng câu trả lời dài dòng, lặp ý hoặc chứa thông tin không cần thiết sẽ bị trừ điểm trực tiếp. Điểm tối đa (5/5) chỉ trao cho câu trả lời vừa đầy đủ ý vừa súc tích.
> 2. **Chấm điểm theo đơn vị thông tin nguyên tử (Atomic Facts / Claims):** Yêu cầu Judge đếm số lượng luận điểm chính xác được đối chiếu với context, thay vì đánh giá cảm tính dựa trên độ dài hay văn phong trau chuốt.
> 3. **Đặt khung độ dài chuẩn (Length guidance):** Nêu rõ độ dài mong muốn trong prompt chấm điểm (ví dụ: "câu trả lời chuẩn thường từ 2–4 câu"), giúp Judge không nhầm lẫn giữa "dài" và "chất lượng cao".

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:*
> 1. **Xác lập Ground Truth và kiểm chứng độ tin cậy:** LLM Judge có thể có thiên kiến cố hữu (self-preference, leniency, severity) hoặc hiểu sai sắc thái chính sách riêng của doanh nghiệp. Nhãn do chuyên gia con người (human labels) là tiêu chuẩn vàng để xác minh LLM Judge có đánh giá đúng hay không.
> 2. **Đo lường mức độ đồng thuận (Inter-Annotator Agreement):** Cần đo hệ số tương quan (Spearman / Pearson hoặc Cohen's Kappa) giữa điểm của LLM Judge và con người. Khi đạt mức đồng thuận cao ($\ge 0.7$), ta mới an tâm đưa LLM Judge vào pipeline tự động.
> 3. **Hiệu chỉnh ngưỡng điểm (Threshold Calibration):** Giúp định nghĩa chính xác mức điểm của LLM (ví dụ 4.0/5.0) tương ứng với hành vi thực tế nào của nhân viên hỗ trợ, tránh trường hợp đặt ngưỡng quá lỏng lẻo hoặc quá khắt khe trong Quality Gate.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | $\ge 0.80$ | Trong domain CSKH OrbitTech, thông tin bịa đặt (hallucination) về giá cả, thời hạn bảo hành hay tiền bồi thường sẽ gây hậu quả pháp lý và thiệt hại tài chính trực tiếp, nên cần ngưỡng nghiêm ngặt nhất. |
| Answer Relevance | $\ge 0.75$ | Đảm bảo câu trả lời luôn đi thẳng vào nhu cầu của người dùng, không trả lời vòng vo hoặc lạc đề làm giảm trải nghiệm khách hàng. |
| Completeness | $\ge 0.70$ | Đảm bảo cung cấp đủ các điều kiện/hướng dẫn cốt lõi; cho phép dung sai nhất định cho các trường hợp câu trả lời tóm tắt ngắn gọn. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> - **Offline Evaluation (Pre-deployment / CI/CD):** Dùng làm Quality Gate tự động chạy trên Golden Dataset (20–100 cases) mỗi khi có Pull Request, đổi prompt, cập nhật model hoặc chỉnh sửa pipeline RAG. Mục đích là phát hiện sớm hồi quy (regression) và chặn code lỗi trước khi merge/deploy.
> - **Online Evaluation (Post-deployment / Production):** Dùng để giám sát liên tục hệ thống đang phục vụ người dùng thật thông qua telemetry (latency, token cost), tín hiệu người dùng (like/dislike, copy, tỷ lệ escalate sang tổng đài viên người), và chạy LLM Judge trên mẫu log định kỳ (sampling) để phát hiện drift trong thế giới thực.
> - **Human Review (Auditing & Improvement Loop):** Dùng định kỳ (hàng tuần/tháng) hoặc khi có khiếu nại nghiêm trọng từ khách hàng. Chuyên gia sẽ xem xét các ca khó, phân tích nguyên nhân gốc (5 Whys), hiệu chỉnh LLM Judge và bổ sung các ca thất bại mới vào Golden Dataset để hoàn thiện vòng lặp Continuous Improvement.

---

## Part 2 — Core Coding (14:45–15:40)

Hoàn thiện các TODO bắt buộc trong `template.py`.

### Task 1 — Data Models

- `QAPair`: question, expected answer, gold context, metadata và retrieved contexts.
- `EvalResult`: answer-side scores, optional retrieval scores, pass/failure fields.
- `overall_score()`: trung bình Faithfulness, Relevance và Completeness.

### Task 2 — RAGASEvaluator

Answer-side:

- `evaluate_faithfulness(answer, context)`
- `evaluate_relevance(answer, question)`
- `evaluate_completeness(answer, expected)`

Retrieval-side:

- `evaluate_context_recall(contexts, expected)`
- `evaluate_context_precision(contexts, expected)`

Full pipeline:

- `run_full_eval(..., contexts=None)` luôn tính ba answer metrics.
- Nếu có `contexts`, tính và lưu thêm Context Recall và Context Precision.
- Retrieval scores không làm thay đổi `overall_score()` và pass rule gốc.

### Task 3 — LLMJudge

- `score_response(question, answer, rubric)`
- `detect_bias(scores_batch)`

### Task 4 — BenchmarkRunner

- `run(qa_pairs, agent_fn, evaluator)`
- `generate_report(results)`
- `run_regression(new_results, baseline_results)`
- `identify_failures(results, threshold)`

`BenchmarkRunner.run()` phải truyền `pair.retrieved_contexts` vào
`run_full_eval()`. Report phải có average của hai retrieval metrics.

### Task 5 — FailureAnalyzer

- `categorize_failures(failures)`
- `find_root_cause(failure)`
- `generate_improvement_suggestions(failures)`
- `generate_improvement_log(failures, suggestions)`

Kiểm tra:

```bash
pytest tests/ -v
```

`rerank_by_overlap()` là TODO bonus của Exercise 3.5. Test tương ứng được skip
nếu bạn chưa làm bonus.

---

## Part 3 — Golden Dataset & Real Benchmark (15:40–16:35)

### Exercise 3.1 — Build the Golden Dataset

Thiết kế và validate dataset theo Mục 5–6 trong `guide_lab.md`. Nội dung 20 QA
được điền trực tiếp trong `golden_dataset.json`; phần dưới chỉ ghi lại kết quả
và quyết định thiết kế, không chép lại toàn bộ QA.

**Kết quả dataset**

| Hạng mục | Kết quả |
|---|---|
| Tổng số records | ____ / 20 |
| Easy | ____ / 5 |
| Medium | ____ / 7 |
| Hard | ____ / 5 |
| Adversarial | ____ / 3 |
| Source documents được sử dụng | ____ / 10 |
| Validator status | PASS / FAIL |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| | | | |
| | | | |
| | | | |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:*

**Xác nhận:**

- [ ] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [ ] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [ ] `python validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

Chạy:

```bash
python domain_assistant.py
python evaluate_answers.py
```

Copy bảng terminal vào đây hoặc điền từ `artifacts/benchmark_results.json`.

| ID | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | | | | | | | | | |
| E02 | | | | | | | | | |
| E03 | | | | | | | | | |
| E04 | | | | | | | | | |
| E05 | | | | | | | | | |
| M01 | | | | | | | | | |
| M02 | | | | | | | | | |
| M03 | | | | | | | | | |
| M04 | | | | | | | | | |
| M05 | | | | | | | | | |
| M06 | | | | | | | | | |
| M07 | | | | | | | | | |
| H01 | | | | | | | | | |
| H02 | | | | | | | | | |
| H03 | | | | | | | | | |
| H04 | | | | | | | | | |
| H05 | | | | | | | | | |
| A01 | | | | | | | | | |
| A02 | | | | | | | | | |
| A03 | | | | | | | | | |

**Aggregate Report**

- Overall pass rate: ____%
- Avg Context Recall: ____
- Avg Context Precision: ____
- Avg Faithfulness: ____
- Avg Relevance: ____
- Avg Completeness: ____
- Failure type distribution: ____

**Ba cases có Overall Score thấp nhất**

1. ID: ____ | Score: ____ | Failure type: ____
2. ID: ____ | Score: ____ | Failure type: ____
3. ID: ____ | Score: ____ | Failure type: ____

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:*

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [ ] Correctness
- [ ] Completeness
- [ ] Relevance
- [ ] Evidence/citation
- [ ] Actionability
- [ ] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | | |
| 4 | | |
| 3 | | |
| 2 | | |
| 1 | | |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| | | |
| | | |
| | | |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:*

### Exercise 3.4 — Framework Comparison (Bonus +5)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: ____ | Framework 2: ____ |
|---|---|---|
| Setup complexity | | |
| Metrics available | | |
| CI/CD integration | | |
| Kết quả trên cùng dataset | | |
| Insight rút ra | | |

- Scores có nhất quán không?
- Framework nào strict hơn và vì sao?
- Hai framework có tìm ra cùng failure cases không?

> *Phân tích:*

### Exercise 3.5 — Retrieval Reranking (Bonus +5)

Mục tiêu: kiểm tra việc đổi thứ tự chunks có tăng Context Precision mà không
thay đổi Context Recall hay không.

1. Chọn ít nhất 5 cases từ `artifacts/actual_answers.json`.
2. Tính Context Recall và Context Precision trước rerank.
3. Implement `rerank_by_overlap()` hoặc một reranker khác.
4. Rerank cùng tập chunks, không thêm hoặc xóa chunk.
5. Tính lại hai metrics và giải thích kết quả.

| ID | Recall before | Recall after | Precision before | Precision after | Delta Precision |
|---|---:|---:|---:|---:|---:|
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| **Avg** | | | | | |

**Tại sao Recall dự kiến không đổi?**

> *Câu trả lời:*

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> *Câu trả lời:*

---

## Part 4 — Reflection (16:35–16:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 16:50–17:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
