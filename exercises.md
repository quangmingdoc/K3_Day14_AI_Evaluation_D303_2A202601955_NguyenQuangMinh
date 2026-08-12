# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 09:15–12:00

**Domain:** Northstar University Student Services

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 09:15–09:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (09:30–09:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Score giảm nhẹ (0.6–0.8) khi answer diễn giải context bằng từ ngữ khác nhưng vẫn đúng ý (paraphrase hợp lý). | Score < 0.6, đặc biệt khi model bịa số liệu, deadline, chính sách không có trong context (hallucination) — rất nguy hiểm cho domain student services vì sai thông tin học vụ/tài chính. | Chặn deploy nếu < 0.6; review từng case bịa đặt, siết prompt "chỉ trả lời dựa trên context", thêm citation requirement. |
| Answer Relevance | 0.6–0.8 khi answer đúng nhưng thừa thông tin phụ (ví dụ trả lời cả điều kiện không liên quan) mà vẫn giải quyết câu hỏi. | < 0.6 khi answer lạc đề hoàn toàn hoặc chỉ trả lời một phần rất nhỏ của câu hỏi multi-part. | Nếu critical: kiểm tra intent detection / query understanding, có thể do retrieval sai chunk dẫn generation lệch hướng. |
| Context Recall | 0.6–0.8 khi thiếu một chi tiết phụ không ảnh hưởng kết luận chính (ví dụ thiếu link tham khảo). | < 0.6 khi thiếu evidence cốt lõi (ví dụ thiếu điều kiện bắt buộc trong quy định) khiến answer dựa trên context không đầy đủ. | Điều tra retriever: tăng top-k, cải thiện chunking, kiểm tra embedding/query rewriting. |
| Context Precision | 0.6–0.8 khi có vài chunk noise xếp sau chunk relevant (không ảnh hưởng nhiều vì LLM vẫn ưu tiên đọc đầu). | < 0.6 khi chunk relevant bị xếp cuối hoặc bị lấn át bởi nhiều chunk không liên quan, dễ gây distraction cho generation. | Cải thiện ranking/reranker, giảm top-k nếu noise nhiều, kiểm tra chunking strategy. |
| Completeness | 0.6–0.8 khi answer thiếu một sub-point nhỏ trong câu hỏi nhiều phần nhưng phần chính đã đủ. | < 0.6 khi answer bỏ sót phần lớn nội dung expected answer, đặc biệt các bước/điều kiện bắt buộc theo quy trình. | Rà soát prompt để enforce trả lời đầy đủ từng phần câu hỏi; kiểm tra completeness heuristic và expected answer coverage. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:* Lấy một cặp answer A/B cho cùng một câu hỏi (một answer chất lượng cao, một chất lượng thấp hơn rõ rệt để có ground-truth). Chạy judge hai lần:
> - **Condition 1:** đưa A trước, B sau (thứ tự A, B).
> - **Condition 2:** đảo thứ tự — B trước, A sau (thứ tự B, A), giữ nguyên nội dung hai answer.
>
> Nếu judge chọn answer xuất hiện trước ở cả hai condition (tức là ở Condition 1 chọn A, ở Condition 2 lại chọn B — answer đứng đầu luôn thắng bất kể nội dung) thì đó là bằng chứng của position bias. Lặp lại trên nhiều cặp câu hỏi và tính tỷ lệ "swap causes flip" để đo mức độ bias định lượng.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:* Rubric cần tách rõ tiêu chí "đúng/đủ nội dung" khỏi "độ dài". Cụ thể:
> - Định nghĩa từng mức điểm (1–5) bằng nội dung bắt buộc phải có (facts, evidence, các bước), không nhắc đến số từ hay độ chi tiết.
> - Thêm chỉ dẫn tường minh cho judge: "Không cộng điểm cho việc answer dài hơn nếu thông tin thêm không liên quan hoặc lặp lại; answer ngắn gọn nhưng đủ ý phải được điểm bằng answer dài dư thừa."
> - Có thể thêm dimension "Conciseness/Actionability" riêng để phạt answer dài dòng không cần thiết, thay vì để độ dài ảnh hưởng ngầm vào điểm Correctness.
> - Test bằng cặp answer cùng nội dung nhưng một bản rút gọn — nếu điểm chênh lệch nhiều thì rubric vẫn còn verbosity bias.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:* LLM judge có thể có bias hệ thống (position, verbosity, self-preference) và cách hiểu rubric khác với con người, nên điểm số của nó không tự động đáng tin. Calibrate bằng cách lấy một tập mẫu đã có human label, so sánh agreement (ví dụ Cohen's kappa hoặc correlation) giữa judge và human. Nếu agreement thấp, cần điều chỉnh rubric/prompt của judge hoặc chọn model judge khác. Việc calibrate định kỳ cũng giúp phát hiện judge bị "drift" theo thời gian khi domain hoặc dữ liệu thay đổi, đảm bảo điểm số CI/CD phản ánh đúng chất lượng thực tế mà người dùng cảm nhận.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.8 | Đây là rủi ro cao nhất — hallucination về học vụ/tài chính có thể gây hậu quả pháp lý/tài chính cho sinh viên, nên phải giữ threshold cao nhất và chặn deploy nếu thấp hơn. |
| Answer Relevance | 0.7 | Answer lạc đề làm giảm trải nghiệm và độ tin cậy, nhưng ít rủi ro trực tiếp hơn faithfulness nên threshold có thể thấp hơn một chút. |
| Completeness | 0.7 | Câu hỏi student services thường nhiều phần (điều kiện, deadline, bước thực hiện); thiếu một phần nhỏ vẫn có thể chấp nhận tạm thời nhưng cần theo dõi sát. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> - **Offline evaluation** (golden dataset, RAGAS metrics chạy trong CI/CD): dùng trước mỗi lần merge/deploy để chặn regression sớm, chi phí thấp, lặp lại được, nhưng chỉ phủ được các case đã biết trước.
> - **Online evaluation** (theo dõi metrics trên traffic thật, A/B test, feedback thumbs up/down): dùng sau khi deploy để phát hiện vấn đề với câu hỏi thực tế mà golden dataset chưa cover, và đo tác động thật đến người dùng.
> - **Human review**: dùng cho các case nhạy cảm (privacy, appeals, tài chính), khi automated score thấp/không rõ ràng (borderline cases), khi cần calibrate LLM judge, hoặc định kỳ audit một sample ngẫu nhiên để đảm bảo automated pipeline vẫn phản ánh đúng chất lượng cảm nhận của người dùng.

---

## Part 2 — Core Coding (09:45–10:40)

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

## Part 3 — Golden Dataset & Real Benchmark (10:40–11:35)

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

Thiết kế rubric domain-specific cho Student Services. Mỗi mức phải đủ cụ thể để
hai người chấm độc lập có thể hiểu giống nhau.

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

### Exercise 3.4 — Framework Comparison (Bonus +10)

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

## Part 4 — Reflection (11:35–11:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 11:50–12:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
