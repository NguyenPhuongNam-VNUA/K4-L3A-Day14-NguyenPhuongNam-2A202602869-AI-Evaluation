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
| Faithfulness | Câu hỏi mang tính xã giao/mở hoặc suy luận thường thức; model trả lời lịch sự không có trong ngữ cảnh nhưng không khẳng định sai bất kỳ thông tin nghiệp vụ/sản phẩm nào. | Model tự bịa đặt (hallucination) chính sách bảo hành, cam kết thời gian đổi trả, hoặc giá bán sai lệch so với tài liệu OrbitTech. | Giảm temperature về 0.0; thắt chặt system prompt ("Chỉ trả lời dựa trên context được cung cấp"); bổ sung câu trích dẫn dẫn chứng (grounding/citation). |
| Answer Relevance | Khách hàng hỏi câu quá ngắn hoặc mơ hồ; model chủ động hỏi lại làm rõ thông tin hoặc liệt kê các lựa chọn dẫn đến lexical overlap với câu hỏi gốc thấp. | Model trả lời vòng vo, lạc đề, lặp lại thông tin không liên quan hoặc từ chối trả lời yêu cầu hợp lệ của khách hàng. | Cải thiện query rewriting/intent classification ở tầng trước; dùng few-shot examples định hướng model trả lời trực diện, đúng trọng tâm. |
| Context Recall | Expected answer chứa các chi tiết phụ/thông tin bổ sung không bắt buộc; câu trả lời thực tế đã đáp ứng đủ thông tin cốt lõi khách cần. | Retriever bỏ sót hoàn toàn điều khoản cốt lõi (ví dụ: điều kiện thời hạn 30 ngày đổi trả), khiến model không có dữ liệu để trả lời đúng. | Tăng top-k retrieval; tối ưu hóa kích thước chunk và chunk overlap; triển khai Hybrid Search (kết hợp Dense Vector Embeddings và Sparse BM25). |
| Context Precision | k retrieval lớn nhằm bao phủ triệt để, retriever lấy thêm các tài liệu liên quan rộng, nhưng generator vẫn chọn lọc đúng thông tin cần thiết. | Chunk chứa thông tin quan trọng nằm ở cuối danh sách (rank thấp) hoặc bị chôn vùi bởi nhiều chunk nhiễu khiến LLM bị "lost in the middle". | Tích hợp module Cross-Encoder Reranker để sắp xếp các chunk có độ liên quan cao nhất lên đầu; lọc bỏ các chunk có relevance score dưới ngưỡng. |
| Completeness | Khách hàng chỉ hỏi xác nhận một chi tiết cụ thể (Yes/No), câu trả lời không cần liệt kê toàn bộ quy trình 5 bước như expected answer mô tả. | Câu hỏi yêu cầu đầy đủ quy trình nhiều bước nhưng model bỏ sót các bước bắt buộc (như giữ nguyên seal, hóa đơn mua hàng), gây hiểu lầm cho khách. | Sử dụng kỹ thuật Query Decomposition (chia nhỏ câu hỏi phức tạp); bổ sung checklist kiểm tra tính đầy đủ của các tiêu chí cần trả lời trong prompt. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*
> - **Thiết kế thực nghiệm (Pairwise Comparison):**
>   - **Condition 1 (Thứ tự ban đầu):** Đưa cặp câu trả lời vào prompt của LLM Judge với `Answer A` ở vị trí 1 (Option A) và `Answer B` ở vị trí 2 (Option B). Ghi nhận lựa chọn của Judge.
>   - **Condition 2 (Đổi chỗ - Swap Order):** Giữ nguyên câu hỏi và rubric đánh giá, hoán đổi vị trí: đưa `Answer B` lên vị trí 1 và `Answer A` xuống vị trí 2. Ghi nhận lựa chọn của Judge.
> - **Đo lường & Kết luận:** Tính tỷ lệ nhất quán (Consistency Rate). Nếu Option ở vị trí 1 luôn được chấm thắng áp đảo bất kể nội dung bên trong là Answer A hay B, chứng tỏ Judge có Position Bias nghiêm trọng.
> - **Biện pháp xử lý:** Áp dụng kỹ thuật Position Swapping (chạy đánh giá cả 2 lượt hoán đổi và chỉ công nhận thắng khi nhất quán ở cả 2 lần, hoặc lấy trung bình điểm số).

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
> - **Quy định rõ ràng về tiêu chí độ dài trong Rubric:** Nêu rõ trong rubric chấm điểm: *"Chất lượng câu trả lời được đo lường bằng tính chính xác, đúng trọng tâm và đầy đủ của thông tin cần thiết; hoàn toàn KHÔNG phụ thuộc vào độ dài câu từ."*
> - **Đưa tiêu chí tính súc tích (Conciseness) vào Rubric:** Thiết lập thang điểm trừ đối với các câu trả lời chứa thông tin thừa, lặp ý hoặc vòng vo không cần thiết.
> - **Cung cấp Anchor Examples (Few-shot calibration):** Đưa ví dụ mẫu cụ thể trong prompt của Judge, minh họa rõ một câu trả lời ngắn gọn, trực diện, đúng trọng tâm nhận điểm tối đa 5/5, trong khi một câu trả lời dài dòng nhưng thiếu ý hoặc pha loãng ý chính chỉ nhận điểm 2/5 hoặc 3/5.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:*
> - LLM Judge không hoàn hảo và thường mắc các thiên kiến hệ thống (như tự ưu tiên văn phong của chính nó - self-preference, xu hướng chấm quá lỏng lẻo - leniency bias hoặc quá khắt khe - severity bias).
> - Việc hiệu chuẩn (calibration) với tập nhãn của chuyên gia con người (tính các chỉ số tương quan như Cohen's Kappa, Spearman's Rank Correlation) giúp xác minh xem LLM Judge có phản ánh đúng tiêu chuẩn đánh giá thực tế của nghiệp vụ hay không, qua đó phát hiện độ lệch điểm để tinh chỉnh rubric và prompt trước khi triển khai làm Quality Gate tự động.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.85 | Trợ lý khách hàng của OrbitTech không được phép bịa đặt thông tin sai lệch về chính sách đổi trả, bảo hành hoặc giá cả, tránh gây rủi ro pháp lý và khiếu nại nghiêm trọng. |
| Answer Relevance | 0.80 | Đảm bảo câu trả lời luôn đi thẳng vào nhu cầu cốt lõi của khách hàng, hạn chế tối đa tình trạng trả lời lạc đề, lảng tránh hoặc phản hồi vô nghĩa. |
| Completeness | 0.75 | Đảm bảo cung cấp đầy đủ các bước thực hiện, điều kiện và chứng từ cần thiết để khách hàng có thể tự xử lý vấn đề thành công mà không phải hỏi đi hỏi lại. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> - **Offline Evaluation:** Dùng trong CI/CD pipeline trước khi deploy (Pre-deployment Gate) khi có thay đổi về model, prompt, hoặc retriever. Chạy trên Golden Dataset cố định (20–100 QA) để phát hiện hồi quy chất lượng (regression drop > 0.05) một cách nhanh chóng, chi phí thấp và an toàn tuyệt đối cho người dùng.
> - **Online Evaluation:** Dùng liên tục khi hệ thống đang vận hành trực tiếp trên Production (Post-deployment Monitoring) để theo dõi hành vi thật qua các proxy metrics: tỷ lệ bấm thumbs up/down của khách hàng, tỷ lệ cuộc hội thoại phải escalate sang nhân viên hỗ trợ thật, latency và tỷ lệ câu trả lời fallback.
> - **Human Review:** Dùng định kỳ theo phương pháp lấy mẫu kiểm toán (Sampling Audit 5–10% các câu có điểm thấp hoặc bị khách khiếu nại), dùng để gán nhãn ground-truth chất lượng cao khi cập nhật Golden Dataset mới, và giải quyết các trường hợp biên (edge cases) phức tạp mà hệ thống tự động chưa xử lý được.

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
