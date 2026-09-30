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
| Faithfulness | Câu đàm thoại xã giao thông thường (chitchat/greetings) hoặc khi assistant diễn đạt lại context bằng từ đồng nghĩa phong phú hơn nhưng không sai lệch sự thật. | Câu hỏi về chính sách bảo hành, hoàn tiền, số tiền phạt hoặc điều kiện trả hàng mà assistant bịa đặt số liệu hoặc điều khoản không có trong context. | Siết chặt system prompt yêu cầu grounded-only; tích hợp citation requirement và hallucination detection filter trước khi trả lời. |
| Answer Relevance | Câu hỏi của khách hàng quá ngắn hoặc mơ hồ ("OrbitTech", "máy tính"), assistant chủ động đưa thêm thông tin tổng quan định hướng. | Khách hỏi chi tiết cụ thể (ví dụ: phí lưu kho trả hàng) nhưng assistant trả lời lạc sang thời gian giao hàng hoặc quảng cáo sản phẩm khác. | Tinh chỉnh prompt tập trung vào user intent; bổ sung bước phân loại ý định (intent classifier/routing) trước khi sinh câu trả lời. |
| Context Recall | Câu hỏi tra cứu dữ kiện đơn giản (factual lookup) chỉ cần 1 tài liệu chính là đủ đáp án, dù các văn bản phụ trợ không được gom hết. | Câu hỏi đa điều kiện/ngoại lệ (multi-hop / exceptions như điều kiện miễn phí đổi trả) nhưng retriever bỏ sót hoàn toàn văn bản chứa điều khoản ngoại lệ. | Tăng top_k retriever, phối hợp BM25 với Dense Vector Embeddings (hybrid search), điều chỉnh chunk size và chunk overlap. |
| Context Precision | Câu hỏi phức tạp cần nhiều thông tin nền tảng, một số chunk rank thấp chứa thông tin bổ sung và không gây nhiễu cho generator. | Chunk chứa bằng chứng cốt lõi bị đẩy xuống cuối (rank 5+), trong khi các vị trí đầu tiên (rank 1-2) toàn là noise khiến generator bị Lost-in-the-Middle. | Triển khai reranker (cross-encoder/lexical reranking); áp dụng ngưỡng tương đồng tối thiểu (similarity cutoff threshold) để lọc rác. |
| Completeness | Người dùng hỏi câu xác nhận nhanh Yes/No và assistant trả lời trực diện, súc tích mà không nhắc lại toàn bộ chính sách dài dòng. | Khách hàng hỏi thủ tục hoàn tiền nhưng assistant chỉ nói được hoàn tiền mà quên cảnh báo điều kiện nguyên hộp và thời hạn 14 ngày. | Thêm few-shot examples hướng dẫn cấu trúc câu trả lời hoàn chỉnh (điều kiện, thủ tục, ngoại lệ, lưu ý); kiểm tra checklist trước khi output. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*
> - **Thiết kế tập mẫu**: Chọn $N=50$ câu hỏi đa dạng và 2 mô hình ứng viên (Model A, Model B) để sinh 50 cặp câu trả lời $(A, B)$.
> - **Condition 1 (Thứ tự gốc - Original Order)**: Đưa vào LLM Judge với định dạng: `Candidate 1: Answer A`, `Candidate 2: Answer B`. Judge đưa ra lựa chọn thắng/thua.
> - **Condition 2 (Đảo vị trí - Swapped Order)**: Đảo ngược vị trí hiển thị: `Candidate 1: Answer B`, `Candidate 2: Answer A`, giữ nguyên system prompt và rubric.
> - **Phân tích kết quả**: Tính tỷ lệ chọn Candidate 1 ở cả hai conditions. Nếu tồn tại Position Bias, tỷ lệ chọn `Candidate 1` sẽ cao vượt trội (ví dụ $>55\%-60\%$) bất kể nội dung bên trong là A hay B.
> - **Khắc phục**: Khi đưa vào production pipeline, thực hiện đánh giá 2 lượt đảo vị trí (swap-and-average) hoặc chỉ chấp nhận kết quả khi cả hai lượt hoán đổi đều đồng thuận.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
> 1. **Bổ sung tiêu chí Conciseness / Length Penalty**: Quy định rõ trong rubric: *"Không chấm điểm cao cho câu trả lời dài nếu chứa thông tin thừa, lặp lại hoặc lan man. Trừ 1–2 điểm nếu độ dài làm loãng trọng tâm câu trả lời."*
> 2. **Chấm điểm theo Checklist đơn vị thông tin (Key Information Units / Atomic Facts)**: Yêu cầu Judge kiểm tra danh sách các dữ kiện bắt buộc thay vì chấm điểm cảm tính trên ấn tượng chung của đoạn văn bản dài.
> 3. **Quy định ràng buộc cấu trúc**: Bắt buộc Judge đánh giá theo cấu trúc: Trực tiếp trả lời $\to$ Bằng chứng/Điều kiện $\to$ Không thêm thông tin phụ trợ không được hỏi.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:*
> - LLM Judge có những thiên kiến riêng (nhạy cảm với cấu trúc câu, self-preference với cùng họ model, leniency bias chấm điểm quá dễ).
> - Calibration đối chiếu điểm số của LLM Judge với tập dữ liệu được gán nhãn bởi chuyên gia con người (Human Ground Truth) nhằm đo lường độ tin cậy qua các chỉ số tương quan như Cohen's Kappa, Spearman hoặc Pearson Correlation.
> - Qua quá trình calibrate, kỹ sư có thể tinh chỉnh rubric, fine-tune prompt chấm điểm và xác định chính xác các ngưỡng cut-off đáng tin cậy trước khi tích hợp LLM Judge làm quality gate tự động trong CI/CD.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.85 | Ngăn chặn hoàn toàn rủi ro ảo giác (hallucination) về chính sách bảo hành, hoàn tiền hoặc thông số kỹ thuật, tránh rủi ro pháp lý và chi phí đền bù. |
| Answer Relevance | 0.75 | Đảm bảo trợ lý đi thẳng vào trọng tâm câu hỏi của khách hàng, hạn chế tình trạng khách hàng ức chế do nhận câu trả lời lạc đề hoặc phải chuyển tiếp lên nhân viên người. |
| Completeness | 0.70 | Đảm bảo câu trả lời chứa đầy đủ các điều kiện tiên quyết, ngoại lệ và hướng dẫn hành động để khách hàng giải quyết được vấn đề trong một lượt tương tác. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> - **Offline Evaluation (Pre-deployment)**: Chạy tự động trong CI/CD pipeline trên bộ Golden Dataset (như 20 QA) mỗi khi có commit thay đổi code, cập nhật prompt hoặc thay đổi retriever/chunking. Mục đích là làm Quality Gate kiểm soát hồi quy (regression testing) trước khi bản build được phép deploy.
> - **Online Evaluation (Post-deployment / Runtime)**: Chạy liên tục trên môi trường production thông qua A/B testing, đánh giá phản hồi người dùng (thumbs up/down, user complaints), đo tỷ lệ giải quyết thành công (resolution rate) và lấy mẫu chạy qua LLM Judge để giám sát chất lượng thực tế theo thời gian thực.
> - **Human Review (Periodic / Strategic)**: Chuyên gia nghiệp vụ rà soát định kỳ (hàng tuần/hàng tháng) trên các mẫu ngẫu nhiên, các trường hợp điểm thấp (low confidence/failure cases) và các câu hỏi mới phát sinh. Dữ liệu này dùng để hiệu chuẩn LLM Judge, cập nhật chính sách và bổ sung mở rộng bộ Golden Dataset.

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
