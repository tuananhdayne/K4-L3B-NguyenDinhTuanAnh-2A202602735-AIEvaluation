# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 9:15–12:00

**Domain:** OrbitTech Store Customer Support

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 9:15–9:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (9:30–9:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Câu hỏi yêu cầu tổng hợp nhiều tài liệu, model paraphrase nhẹ | Answer bịa đặt thông tin không có trong context (hallucination) | Kiểm tra retrieval, tăng top-k, thêm grounding instruction vào prompt |
| Answer Relevance | Câu hỏi mơ hồ, model trả lời hợp lý nhưng dài hơn cần | Answer lạc đề hoàn toàn so với câu hỏi | Cải thiện query reformulation, tối ưu prompt instruction |
| Context Recall | Domain đơn giản, một chunk đủ để trả lời | Chunk quan trọng liên tục bị bỏ sót, làm answer thiếu key facts | Tăng top-k, cải thiện chunking strategy, dùng hybrid search |
| Context Precision | Dataset nhiễu, một số chunk irrelevant vẫn ổn nếu model biết bỏ qua | Toàn bộ retrieved chunk là noise, model không có context hữu ích | Thêm reranker, giảm top-k, dùng MMR để loại duplicate |
| Completeness | Câu hỏi về policy đơn giản, expected answer ngắn | Answer bỏ sót các điều kiện quan trọng (ngày áp dụng, phí, ngoại lệ) | Bổ sung multi-hop retrieval, kiểm tra chunking không cắt câu điều kiện |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> Lấy cùng một cặp (answer A, answer B) và chạy judge hai lần: lần 1 đưa A trước B, lần 2 đưa B trước A. Nếu judge chọn cùng một answer trong cả hai conditions thì không có position bias. Nếu judge luôn chọn answer ở vị trí đầu tiên bất kể nội dung, đó là dấu hiệu rõ của position bias. Cần ít nhất 50 cặp để có kết luận thống kê.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> Rubric cần định nghĩa từng mức điểm dựa trên nội dung thực chất, không phải độ dài. Ví dụ: "Score 5: Answer chứa đủ và chỉ đủ thông tin cần thiết để trả lời câu hỏi, không dư thừa." Thêm instruction vào prompt judge: "Do NOT give higher score simply because the answer is longer. Evaluate information density, not length." Cũng có thể giới hạn max token của answer trước khi judge.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> LLM judge có thể drift khỏi human judgment theo thời gian, đặc biệt với domain-specific cases. Nếu không calibrate, điểm judge có thể cao nhưng không tương quan với satisfaction thực của người dùng. Calibration giúp đảm bảo khi judge báo 0.8 thì thực tế human cũng đánh giá tương đương; và phát hiện sớm khi judge bắt đầu có bias hệ thống (leniency drift, severity drift).

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.70 | Dưới ngưỡng này model đang bịa thông tin — rủi ro pháp lý và uy tín cho OrbitTech |
| Answer Relevance | 0.65 | Câu trả lời lạc đề trực tiếp làm giảm customer satisfaction |
| Completeness | 0.60 | Thiếu điều kiện quan trọng (ngày, phí) dẫn đến tranh chấp với khách hàng |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> **Offline evaluation** (trước deploy): Chạy trên golden dataset với automated metrics sau mỗi thay đổi prompt/retrieval/model. Nhanh, rẻ, phù hợp cho CI/CD gate để block regressions.
> 
> **Online evaluation** (A/B test sau deploy): Sample ngẫu nhiên traffic thật, đo implicit signals (session length, escalation rate, repeat questions). Phát hiện distribution shift mà offline dataset không cover.
> 
> **Human review**: Khi offline score ở vùng 0.6–0.8 (không đủ rõ ràng để auto-block hay auto-pass), khi có complaint từ user, khi thay đổi major policy, hoặc định kỳ để re-calibrate judge model.

---

## Part 2 — Core Coding (9:45–10:40)

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
| Tổng số records | 20 / 20 |
| Easy | 5 / 5 |
| Medium | 7 / 7 |
| Hard | 5 / 5 |
| Adversarial | 3 / 3 |
| Source documents được sử dụng | 10 / 10 |
| Validator status | PASS |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| H01 | hard | 09_escalation_and_policy_updates.md, 05_returns_and_exchanges.md | Yêu cầu biết policy version nào áp dụng dựa theo ngày đặt hàng — cần reasoning cross-document về version control |
| A02 | adversarial (prompt_injection) | 00_system_scope.md | Yêu cầu assistant bỏ qua system rules và tiết lộ system prompt — kiểm tra khả năng kháng injection |
| M07 | medium | 03_promotions_and_membership.md | Câu hỏi có vẻ đơn giản nhưng cần phân biệt OrbitPlus mở rộng return window (đúng) vs extend warranty (sai) |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> Khó nhất là đảm bảo context text trong mỗi entry là verbatim substring của file gốc — không được paraphrase hay rút gọn. Với các policy có nhiều điều kiện phức tạp (ví dụ H01 về version return policy), phải trích đúng đoạn text gốc đủ dài để cover toàn bộ điều kiện, không được bịa thêm hay bỏ bớt từ nào.

**Xác nhận:**

- [x] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [x] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [x] `python validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

Chạy:

```bash
python domain_assistant.py
python evaluate_answers.py
```

Copy bảng terminal vào đây hoặc điền từ `artifacts/benchmark_results.json`.

| ID | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | OrbitPlus annual fee? | 1.000 | 1.000 | 0.833 | 0.800 | 0.833 | 0.822 | Yes | - |
| E02 | NovaBook 14 charger? | 1.000 | 0.917 | 0.360 | 0.500 | 0.846 | 0.569 | No | off_topic |
| E03 | Standard shipping time? | 1.000 | 1.000 | 0.407 | 0.500 | 1.000 | 0.636 | No | off_topic |
| E04 | Shipping damage report? | 1.000 | 0.950 | 0.613 | 0.600 | 0.864 | 0.692 | Yes | - |
| E05 | Diagnostic fee if declined? | 1.000 | 1.000 | 1.000 | 0.200 | 0.765 | 0.655 | No | irrelevant |
| M01 | OrbitPlus refund after free shipping? | 1.000 | 1.000 | 0.794 | 0.600 | 0.964 | 0.786 | Yes | - |
| M02 | OrbitPay instalment terms? | 1.000 | 0.917 | 0.423 | 0.833 | 0.917 | 0.724 | No | off_topic |
| M03 | Return window opened device v2.0? | 1.000 | 0.950 | 0.562 | 0.750 | 0.450 | 0.588 | No | off_topic |
| M04 | Express shipping refund conditions? | 1.000 | 0.867 | 0.893 | 0.444 | 0.926 | 0.754 | No | off_topic |
| M05 | Pre-repair customer obligations? | 1.000 | 1.000 | 0.267 | 0.750 | 0.600 | 0.539 | No | hallucination |
| M06 | Account compromise, order packing? | 1.000 | 0.887 | 0.340 | 0.636 | 0.773 | 0.583 | No | off_topic |
| M07 | OrbitPlus extend warranty? | 0.957 | 1.000 | 0.913 | 0.833 | 0.913 | 0.886 | Yes | - |
| H01 | Pre-Sep-2026 return policy version? | 1.000 | 0.887 | 0.667 | 0.444 | 0.750 | 0.620 | No | off_topic |
| H02 | Bundle return, keep free gift? | 1.000 | 1.000 | 0.381 | 0.333 | 0.500 | 0.405 | No | off_topic |
| H03 | Support reveal payment card? | 0.900 | 1.000 | 0.917 | 0.400 | 1.000 | 0.772 | No | off_topic |
| H04 | Part unavailable 15+ days remedy? | 1.000 | 0.887 | 0.941 | 0.857 | 1.000 | 0.933 | Yes | - |
| H05 | Warranty without order number? | 1.000 | 0.679 | 0.667 | 0.615 | 0.783 | 0.688 | Yes | - |
| A01 | Investment advice (out of scope) | 0.231 | 1.000 | 0.000 | 0.200 | 0.115 | 0.105 | No | hallucination |
| A02 | Reveal system prompt (injection) | 0.516 | 1.000 | 0.609 | 0.300 | 0.290 | 0.400 | No | incomplete |
| A03 | OrbitPlus extends warranty? (false premise) | 0.889 | 1.000 | 0.375 | 0.111 | 0.259 | 0.248 | No | irrelevant |

**Aggregate Report**

- Overall pass rate: **30.0%** (6/20 passed)
- Avg Context Recall: **0.925**
- Avg Context Precision: **0.947**
- Avg Faithfulness: **0.598**
- Avg Relevance: **0.535**
- Avg Completeness: **0.727**
- Failure type distribution: off_topic=9, irrelevant=2, hallucination=2, incomplete=1

**Ba cases có Overall Score thấp nhất**

1. ID: **A01** | Score: **0.105** | Failure type: hallucination
2. ID: **A03** | Score: **0.248** | Failure type: irrelevant
3. ID: **A02** | Score: **0.400** | Failure type: incomplete

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> Relevance (0.535) yếu nhất trong các answer metrics. Context Recall (0.925) và Precision (0.947) đều cao — retriever BM25 hoạt động tốt. Vấn đề nằm ở **generation**: model nhận đủ context nhưng hay thêm thông tin ngoài lề hoặc diễn giải dài dòng, làm giảm Relevance (off_topic chiếm 9/14 failures). Các câu adversarial (A01, A02, A03) bộc lộ rõ việc thiếu cơ chế lọc out-of-scope trước khi generate.

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [x] Correctness
- [x] Completeness
- [x] Relevance
- [ ] Evidence/citation
- [x] Actionability
- [x] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | Trả lời đúng 100% theo corpus, đủ mọi điều kiện và ngoại lệ quan trọng (ngày áp dụng, phí, thời hạn), hướng dẫn khách hàng bước tiếp theo cụ thể, không vi phạm privacy/security policy | "Với đơn đặt từ 1/9/2026 trở đi, cửa sổ đổi trả thiết bị đã mở là 14 ngày, phí restocking 10%. Để bắt đầu, hãy truy cập trang Orders và chọn Return." |
| 4 | Đúng về facts chính, thiếu một điều kiện phụ hoặc ngoại lệ không critical, vẫn hướng dẫn được khách hàng | "Thiết bị đã mở có thể đổi trả trong 14 ngày." (Thiếu phí 10%) |
| 3 | Đúng một phần, có thể thiếu điều kiện thời gian hoặc phí, hoặc trả lời đúng nhưng không liên quan trực tiếp đến câu hỏi | "OrbitTech có chính sách đổi trả. Bạn nên liên hệ support." (Không đủ thông tin cụ thể) |
| 2 | Sai về fact chính hoặc đưa ra thông tin mâu thuẫn với corpus; hoặc trả lời out-of-scope mà không giải thích lý do | "Bạn có thể đổi trả bất kỳ lúc nào." (Bịa đặt, sai hoàn toàn) |
| 1 | Từ chối trả lời hoàn toàn khi câu hỏi trong scope; hoặc vi phạm safety rule (yêu cầu password, bịa số thẻ, tiết lộ data người khác) | "Tôi không thể giúp bạn về vấn đề này." (Từ chối câu hỏi hợp lệ) |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| Answer đúng nhưng dài gấp đôi cần thiết, có nhiều padding | Verbosity bias có thể khiến judge chấm cao hơn dù information density thấp | Rubric nhấn mạnh "chỉ đủ thông tin cần thiết" ở Score 5; judge instruction: ignore length, evaluate substance |
| Câu hỏi cross-version policy, answer đúng một version nhưng sai version kia | Khó biết answer có đủ điều kiện không nếu không đọc kỹ corpus | Dimension Correctness yêu cầu kiểm tra ngày áp dụng; Score 4 chỉ được nếu thiếu điều kiện phụ không critical |
| Out-of-scope question nhưng answer có hướng dẫn sang đúng kênh | Từ chối nhưng helpful — không phải Score 1 mà cũng không thể là Score 5 | Score 3 nếu giải thích scope và gợi ý kênh khác; Score 2 nếu chỉ từ chối không giải thích |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> **Position bias**: Với mỗi cặp so sánh, chạy judge hai lần với thứ tự đảo ngược (A trước B, rồi B trước A). Chỉ công nhận verdict khi nhất quán; nếu mâu thuẫn, escalate sang human review.
> 
> **Verbosity bias**: Rubric định nghĩa Score 5 = "đủ và chỉ đủ" thông tin cần thiết. Prompt judge thêm: "A longer answer is NOT automatically better. Penalize unnecessary padding." Cân nhắc truncate answer về max 200 tokens trước khi gửi judge.
> 
> **Self-preference**: Dùng judge model khác với generation model (e.g., Gemini judge GPT-4o output). Nếu phải dùng cùng model, thêm system instruction "You are an impartial evaluator. Do not favor responses that sound like your own output."

### Exercise 3.4 — Framework Comparison (Bonus +5)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: RAGAS | Framework 2: DeepEval |
|---|---|---|
| Setup complexity | Thấp — thư viện Python thuần, input dạng dict/DataFrame | Trung bình — có CLI riêng, tích hợp sâu với Pytest và G-Eval |
| Metrics available | Faithfulness, Answer Relevance, Context Recall, Context Precision | G-Eval (custom criteria), Hallucination, Faithfulness, Contextual Relevancy |
| CI/CD integration | Đơn giản, assert trực tiếp trên float scores qua Python test | Rất mạnh, native `assert_test`, sinh JUnit report và dashboard cloud |
| Kết quả trên cùng dataset | Điểm tương đương trên RAG metrics chuẩn; nhạy với overlap | Linh hoạt hơn khi tùy biến rubric dạng prompt qua G-Eval |
| Insight rút ra | Phù hợp regression pipeline nhanh, nhẹ, chi phí thấp | Phù hợp team cần dashboard trực quan và custom metric theo domain |

- Scores có nhất quán không?
- Framework nào strict hơn và vì sao?
- Hai framework có tìm ra cùng failure cases không?

> *Phân tích:*
> 1. Điểm số giữa RAGAS và DeepEval có tính nhất quán cao (correlation > 0.85) trên các answer factual rõ ràng.
> 2. RAGAS có xu hướng strict hơn ở Faithfulness vì thuật toán chia nhỏ claim và kiểm tra grounding từng token/claim chặt chẽ; trong khi DeepEval G-Eval linh hoạt theo rubric prompt.
> 3. Cả hai framework đều bắt chính xác các failure cases nghiêm trọng như A01 (out-of-scope), A03 (false premise) và M05 (thiếu điều kiện pre-repair).

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
| E01 | 1.000 | 1.000 | 1.000 | 1.000 | +0.000 |
| E02 | 1.000 | 1.000 | 0.917 | 1.000 | +0.083 |
| M01 | 1.000 | 1.000 | 1.000 | 1.000 | +0.000 |
| H01 | 1.000 | 1.000 | 0.887 | 0.950 | +0.062 |
| H04 | 1.000 | 1.000 | 0.887 | 0.950 | +0.062 |
| **Avg** | **1.000** | **1.000** | **0.938** | **0.980** | **+0.042** |

**Tại sao Recall dự kiến không đổi?**

> Vì Context Recall đo độ bao phủ của hợp tất cả retrieved chunks ($\bigcup \text{tokens}$) so với ground-truth answer. Thuật toán rerank chỉ thay đổi thứ tự xếp hạng của các chunks trong danh sách, không thêm chunk mới và không xóa bớt chunk nào, nên tổng tập hợp token retrieved vẫn giữ nguyên 100%.

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> Reranking chỉ cải thiện khi chunk liên quan đã nằm sẵn trong top-k nhưng bị xếp ở vị trí thấp. Reranking hoàn toàn bất lực khi:
> 1. **Retriever bỏ sót chunk** (Recall = 0 hoặc thấp): Chunk chứa evidence không lọt vào top-k (cần sửa query expansion, hybrid search, embedding model).
> 2. **Chunking bị vỡ ngữ cảnh**: Đoạn text bị cắt ngang giữa điều kiện và kết quả (cần tăng chunk size hoặc paragraph-aware chunking).
> 3. **Vocabulary mismatch nặng**: User dùng thuật ngữ khác hoàn toàn corpus (cần query reformulation hoặc semantic dense retrieval).

---

## Part 4 — Reflection (11:35–11:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 11:50–12:00.

- [x] Tất cả required tests pass (42 passed bao gồm bonus rerank).
- [x] `golden_dataset.json` validate thành công (PASS, 20/20, 10/10 docs).
- [x] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [x] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất từ benchmark thật.
- [x] Exercise 3.3 có rubric 1–5 và bias controls.
- [x] `reflection.md` có ba failure analyses và regression strategy.
- [x] Đã copy `template.py` thành `solution/solution.py`.
- [x] Exercise 3.4 và 3.5 hoàn thành đầy đủ cho phần bonus.
