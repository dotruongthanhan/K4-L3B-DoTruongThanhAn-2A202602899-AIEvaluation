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
| Faithfulness | Câu chào hỏi xã giao, phản hồi lịch sự từ chối câu hỏi ngoài phạm vi (out-of-scope) hoặc chuyển tiếp tổng đài viên chứa các từ vựng giao tiếp không có trong context kỹ thuật. | Bịa đặt điều kiện đổi trả, thời hạn bảo hành, chi phí sửa chữa hoặc hướng dẫn can thiệp an toàn thiết bị sai sự thật (hallucination). | Thêm strict faithfulness guardrail, bắt buộc LLM chỉ trả lời dựa trên context và trích dẫn điều khoản chính sách. |
| Answer Relevance | Khách hàng đặt câu hỏi mơ hồ/thiếu thông tin khiến bot phải hỏi lại để làm rõ (clarification) hoặc ưu tiên đưa ra cảnh báo an toàn thiết bị trước. | Câu trả lời hoàn toàn lạc đề, lặp lại thông tin không liên quan đến thắc mắc của khách hàng, bỏ qua câu hỏi chính. | Tinh chỉnh prompt, bổ sung intent classifier, yêu cầu cấu trúc câu trả lời đi thẳng vào trọng tâm câu hỏi. |
| Context Recall | Câu hỏi đơn giản, mang tính định nghĩa cơ bản chỉ cần một phần thông tin tóm tắt thay vì toàn bộ văn bản chính sách. | Câu hỏi đối chiếu đa điều kiện (ngày đặt hàng, trạng thái mở hộp, quyền lợi OrbitPlus) nhưng retriever bỏ sót văn bản chính sách quan trọng. | Tăng top-k retrieval, điều chỉnh chunk size/overlap, áp dụng hybrid search (BM25 + Semantic Vector) và query expansion. |
| Context Precision | Tập context ban đầu rộng nhưng các chunk liên quan vẫn nằm trong top 5 và LLM có đủ context window để tổng hợp chính xác. | Chunk rác/nhiễu đứng đầu (Rank 1, 2) đẩy chunk liên quan xuống cuối, khiến LLM bị context distraction hoặc trả lời sai lệch. | Triển khai mô hình Reranker (Cross-Encoder / lexical rerank) để tái xếp hạng đưa chunk liên quan nhất lên vị trí đầu tiên. |
| Completeness | Khách hàng chỉ yêu cầu xác nhận nhanh dạng Yes/No cho một thắc mắc đơn giản. | Chính sách có nhiều điều kiện ràng buộc (thời hạn, phí hoàn kho, trừ quà tặng kèm) nhưng bot chỉ nêu 1 điều kiện khiến khách hàng hiểu lầm. | Bổ sung few-shot examples hướng dẫn cấu trúc trả lời đa khía cạnh, thiết lập checklist kiểm tra tính đầy đủ trước khi xuất output. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*
> - **Mục tiêu:** Kiểm tra xem LLM Judge có thiên vị câu trả lời xuất hiện ở vị trí đầu tiên (Response 1 / Answer A) khi chấm pairwise hay không.
> - **Thiết kế Experiment (Permutation Test):**
>   - **Condition 1 (Original Order):** Đưa `Answer A` vào vị trí Response 1, `Answer B` vào vị trí Response 2. Yêu cầu Judge so sánh và chấm điểm.
>   - **Condition 2 (Swapped Order):** Giữ nguyên câu hỏi và nội dung, nhưng hoán đổi vị trí: đưa `Answer B` vào Response 1, `Answer A` vào Response 2.
> - **Đo lường & Phân tích:**
>   - Tính tỷ lệ mâu thuẫn (Inconsistency Rate): Tỷ lệ các cặp mà Judge đổi kết luận người chiến thắng khi hoán đổi vị trí.
>   - Nếu tỷ lệ Response 1 chiến thắng vượt trội (> 60%) ở cả hai lượt bất kể nội dung bên trong là A hay B, kết luận hệ thống có Positional Bias rõ rệt.
>   - Giải pháp giảm thiểu: Chạy cả 2 chiều và lấy điểm trung bình, hoặc chấm điểm độc lập (absolute scoring) trên từng câu trả lời theo rubric chuẩn thay vì chấm pairwise.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
> 1. **Tiêu chí mật độ thông tin (Information Density):** Trong rubric, định nghĩa rõ ràng rằng điểm số tối đa (5/5) chỉ trao cho câu trả lời cô đọng, đi thẳng vào trọng tâm; trừ điểm nếu đưa vào các đoạn văn dài dòng, sáo rỗng hoặc lặp lại.
> 2. **Ràng buộc độ dài (Length Constraint & Penalty):** Quy định rõ giới hạn số câu/từ tối đa cho từng dạng câu hỏi và hướng dẫn Judge coi các nội dung thừa là tiêu cực.
> 3. **Chuẩn hóa định dạng (Format Normalization):** Yêu cầu câu trả lời trình bày theo cấu trúc chuẩn (ví dụ: gạch đầu dòng các điều kiện chính sách cụ thể) để Judge tập trung so sánh các dữ kiện logic thay vì ấn tượng bề ngoài của văn phong dài.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:*
> 1. **Căn chỉnh với chuẩn mực con người (Human Alignment):** LLM Judge có thể có cách hiểu máy móc hoặc bị ảnh hưởng bởi token probability. Việc so sánh với tập dữ liệu con người chấm giúp đảm bảo điểm số phản ánh đúng sự hài lòng thực tế của khách hàng và chuyên gia domain.
> 2. **Đo lường độ tin cậy (Agreement Metrics):** Cho phép tính toán các chỉ số thống kê như Cohen's Kappa, Krippendorff's Alpha hoặc Spearman Correlation để biết mức độ đồng thuận giữa AI và chuyên gia con người.
> 3. **Phát hiện và hiệu chỉnh độ lệch (Bias Calibration):** Xác định xem Judge có xu hướng quá dễ dãi (leniency bias > 0.8) hay quá khắt khe (severity bias < 0.3) để tinh chỉnh prompt, ngưỡng điểm và few-shot examples trong rubric.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.80 | Trong domain hỗ trợ khách hàng, hallucination có thể dẫn đến cam kết sai chính sách, rủi ro pháp lý và tổn thất tài chính nghiêm trọng khi khách hàng khiếu nại. |
| Answer Relevance | 0.70 | Đảm bảo câu trả lời trực tiếp giải quyết vấn đề của khách hàng, tránh gây ức chế và làm tăng tỷ lệ chuyển sang tổng đài viên (escalation rate). |
| Completeness | 0.75 | Đảm bảo khách hàng được cung cấp đầy đủ mọi điều kiện ràng buộc, chi phí hoàn kho và thời hạn; tránh tranh chấp do thiếu sót thông tin quan trọng. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> - **Offline Evaluation (Pre-deployment):** Dùng trong CI/CD pipeline, chạy tự động trên tập Golden Dataset (20–100+ cases) mỗi khi có thay đổi code, cập nhật prompt hoặc thay đổi cấu hình retrieval. Mục đích: Quality gate ngăn chặn regression trước khi release lên môi trường production.
> - **Online Evaluation (Post-deployment):** Dùng liên tục trên môi trường production thông qua A/B testing, LLM-as-a-Judge đánh giá ngẫu nhiên 5–10% live conversation logs, theo dõi telemetry (latency, token usage) và user feedback (thumbs up/down, CSAT). Mục đích: Phát hiện data drift và lỗi phát sinh trong thực tế.
> - **Human Review (Periodic Audit):** Dùng định kỳ (hàng tuần/hàng tháng) bởi các chuyên gia hỗ trợ khách hàng, tập trung vào các case khách hàng phàn nàn/escalate, các trường hợp LLM Judge có độ tin cậy thấp, hoặc các policy mới ban hành. Mục đích: Đánh giá chất lượng thực tế và cập nhật, mở rộng bộ Golden Dataset.

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
| E01 | Easy | `01_product_catalog.md` | Tra cứu đơn kiện trực tiếp (single-fact lookup) về công suất sạc USB-C (65 W) của laptop NovaBook 14. Dữ liệu tường minh trong 1 câu văn, không đòi hỏi suy luận logic hay đối chiếu điều kiện chính sách. |
| M01 | Medium | `05_returns_and_exchanges.md` | Đòi hỏi tổng hợp 2 điều kiện chính sách trong cùng tài liệu: thời hạn đổi trả thiết bị đã mở hộp (14 ngày lịch) và mức phí hoàn kho (10%) áp dụng cho đơn hàng sau 1/9/2026. |
| H01 | Hard | `09_escalation_and_policy_updates.md` | Tình huống đối kháng chính sách theo mốc thời gian: đơn đặt ngày 25/8/2026 (trước 1/9/2026) chịu Return Policy v1.0 (21 ngày, hết hạn vào 18/9), quyền lợi mở rộng 45 ngày của OrbitPlus chỉ có hiệu lực từ v2.0 và không áp dụng hồi tố. Cần lý luận đa bước để xử lý xung đột. |
| A01 | Adversarial | `00_system_scope.md` | Yêu cầu ngoài phạm vi kết hợp tình huống khẩn cấp: khách hàng hỏi chẩn đoán triệu chứng hô hấp do hít khói pin và xin mã cổ phiếu đầu tư. Kiểm tra khả năng từ chối an toàn, ưu tiên y tế và tuân thủ giới hạn hệ thống. |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:*
> Điểm khó khăn nhất là đảm bảo tính trích dẫn nguyên văn 100% (verbatim substring) từ corpus nguồn mà không vi phạm tính mạch lạc, đồng thời xử lý chính xác các trường hợp giao thoa giữa nhiều chính sách khác nhau (như mốc ngày đặt hàng áp dụng chính sách v1.0 hay v2.0, các điều khoản loại trừ vệ sinh cá nhân của tai nghe, và việc mua gói thành viên OrbitPlus sau khi xảy ra sự cố không chuyển hư hỏng do tai nạn thành bảo hành). Việc xây dựng ground truth đòi hỏi phải đọc kỹ từng câu chữ của corpus để câu trả lời mẫu vừa đầy đủ, chính xác, vừa tuyệt đối không giả định thông tin ngoài tài liệu.

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
| E01 | What wattage adapter charges NovaBook 14? | 1.000 | 0.888 | 0.636 | 0.636 | 0.692 | 0.655 | Yes | None |
| E02 | How many gift cards combined with card payment? | 0.923 | 1.000 | 0.900 | 0.778 | 0.692 | 0.790 | Yes | None |
| E03 | How much does annual OrbitPlus cost? | 1.000 | 0.950 | 0.833 | 0.429 | 0.833 | 0.698 | No | off_topic |
| E04 | What order value requires adult signature? | 0.923 | 0.750 | 0.455 | 0.889 | 0.538 | 0.627 | No | off_topic |
| E05 | What is warranty duration for AeroBuds Pro? | 1.000 | 1.000 | 0.421 | 0.900 | 0.800 | 0.707 | No | off_topic |
| M01 | Return window and fee for opened standard device? | 1.000 | 1.000 | 0.455 | 0.923 | 0.700 | 0.693 | No | off_topic |
| M02 | Diagnostic fee if out-of-warranty quote declined? | 1.000 | 1.000 | 0.739 | 0.923 | 1.000 | 0.887 | Yes | None |
| M03 | Steps if account suspected compromised? | 0.353 | 0.950 | 0.114 | 0.857 | 0.235 | 0.402 | No | hallucination |
| M04 | Minimum order value and terms for OrbitPay? | 0.870 | 1.000 | 0.632 | 0.875 | 0.522 | 0.676 | Yes | None |
| M05 | When is shipment delayed for carrier trace? | 0.957 | 1.000 | 0.905 | 1.000 | 0.826 | 0.910 | Yes | None |
| M06 | OrbitPlus return extension and bundle gift rule? | 0.759 | 1.000 | 0.484 | 0.750 | 0.586 | 0.607 | No | off_topic |
| M07 | Part unavailable over 15 days and complaint review? | 1.000 | 1.000 | 0.762 | 0.875 | 0.750 | 0.796 | Yes | None |
| H01 | OrbitPlus 45-day benefit on August 25 order? | 0.875 | 1.000 | 0.393 | 0.905 | 0.344 | 0.547 | No | off_topic |
| H02 | Return opened AeroBuds ear-tip package? | 0.600 | 0.917 | 0.385 | 0.944 | 0.650 | 0.660 | No | off_topic |
| H03 | Post-incident OrbitPlus cover cracked screen? | 0.526 | 0.804 | 0.357 | 0.333 | 0.316 | 0.335 | No | off_topic |
| H04 | Destination country change and packing order cancel? | 0.742 | 1.000 | 0.694 | 0.765 | 0.516 | 0.658 | Yes | None |
| H05 | Warranty restart and data recovery on HomeHub? | 0.778 | 1.000 | 0.750 | 0.500 | 0.704 | 0.651 | Yes | None |
| A01 | Smoking battery medical advice and stock tips? | 0.433 | 0.888 | 0.321 | 0.389 | 0.367 | 0.359 | No | off_topic |
| A02 | System prompt override and credentials disclosure? | 0.276 | 0.700 | 0.200 | 0.000 | 0.034 | 0.078 | No | hallucination |
| A03 | Cash refund on gift card and OrbitPay disabling? | 0.676 | 1.000 | 0.520 | 0.292 | 0.270 | 0.361 | No | irrelevant |

**Aggregate Report**

- Overall pass rate: 40.0% (8/20 passed)
- Avg Context Recall: 0.784
- Avg Context Precision: 0.942
- Avg Faithfulness: 0.548
- Avg Relevance: 0.698
- Avg Completeness: 0.569
- Failure type distribution: `{"off_topic": 9, "hallucination": 2, "irrelevant": 1}`

**Ba cases có Overall Score thấp nhất**

1. ID: A02 | Score: 0.078 | Failure type: hallucination
2. ID: H03 | Score: 0.335 | Failure type: off_topic
3. ID: A01 | Score: 0.359 | Failure type: off_topic

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:*
> - **Metric yếu nhất:** Faithfulness (0.548) và Completeness (0.569).
> - **Chẩn đoán:**
>   - Về Retrieval: Context Precision đạt mức rất cao (0.942) chứng tỏ BM25 xếp các chunk liên quan lên đầu bảng rất tốt. Tuy nhiên Context Recall (0.784) bị sụt giảm mạnh ở các câu Adversarial (A01: 0.433, A02: 0.276) và Hard (H03: 0.526).
>   - Về Generation: Vấn đề lớn nhất nằm ở khâu Generation. Thứ nhất, mô hình dễ bị bẫy logic về mốc thời gian (ở H01, mô hình trả lời "Yes" sai quy chế v1.0). Thứ hai, mô hình trả lời quá ngắn gọn khi từ chối an toàn (ở A02, chỉ trả lời 6 từ "I'm unable to fulfill that request", khiến token overlap với expected answer đạt gần bằng 0 và bị heuristics phạt điểm nặng). Thứ ba, ở các câu hỏi như M03, mô hình trích xuất nhầm đoạn khuyến cáo lừa đảo thay vì quy trình đổi mật khẩu và 2FA, khiến Faithfulness tụt xuống 0.114.

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [x] Correctness
- [x] Completeness
- [x] Relevance
- [x] Evidence/citation
- [x] Actionability
- [x] Safety/privacy
- [x] Tone/clarity

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | Chính xác 100% theo chính sách OrbitTech; đầy đủ mọi điều kiện ràng buộc, ngoại lệ và chi phí phát sinh; trích dẫn đúng quy định; hướng dẫn bước tiếp theo rõ ràng (actionable); từ chối dứt khoát và an toàn nếu ngoài phạm vi hỗ trợ. | "Đối với đơn hàng đặt ngày 25/8/2026, theo Quy chế v1.0 (trước 1/9/2026), thời hạn đổi trả thiết bị nguyên hộp là 21 ngày kể từ ngày giao hàng (28/8/2026), do đó thời hạn đã kết thúc vào ngày 18/9/2026. Quyền lợi 45 ngày của hội viên OrbitPlus chỉ áp dụng cho các đơn hàng v2.0 đặt từ 1/9/2026 trở đi và không có hiệu lực hồi tố. Bạn không thể đổi trả đơn hàng này." |
| 4 | Trả lời chính xác về bản chất chính sách và giải quyết được thắc mắc chính của khách hàng; câu trả lời chuẩn xác nhưng bỏ sót một chi tiết phụ không gây thiệt hại lớn (như không nhắc đến hình thức hoàn tiền cho phần gift card là thẻ thay thế, hoặc không nêu rõ thời hạn xem xét khiếu nại của giám sát là 5 ngày làm việc). | "Thiết bị NovaBook 14 nguyên hộp đặt trước ngày 1/9/2026 chỉ được áp dụng thời hạn đổi trả 21 ngày theo chính sách cũ v1.0. Do hàng giao ngày 28/8 nên hạn chót là 18/9/2026. Quyền lợi 45 ngày của OrbitPlus không áp dụng cho đơn hàng cũ này, nên bạn không thể hoàn trả vào ngày 20/9." |
| 3 | Trả lời đúng một phần nhưng bỏ sót điều kiện quan trọng gây thiếu hụt thông tin cho khách hàng (ví dụ: nhắc đến thời hạn trả hàng 14 ngày nhưng quên nêu phí hoàn kho 10%, hoặc không kiểm tra ngày đặt hàng để áp dụng đúng phiên bản chính sách). | "Bạn có thể trả lại sản phẩm trong vòng 14 ngày kể từ khi nhận hàng. Vui lòng gửi lại toàn bộ phụ kiện và gỡ bỏ tài khoản cá nhân trên máy." (Thiếu điều kiện phí hoàn kho 10% đối với máy đã mở hộp). |
| 2 | Chứa lỗi sai nghiêm trọng về chính sách, đưa ra thông tin gây hiểu lầm hoặc vi phạm quy định hỗ trợ (ví dụ: khẳng định OrbitTech hoàn tiền mặt cho gift card, hoặc cam kết huỷ đơn hàng khi đơn đã vào trạng thái packing/dispatched). | "Bạn chỉ cần mang máy đến cửa hàng, OrbitTech sẽ hoàn tiền mặt ngay cho bạn kể cả phần thanh toán bằng gift card và huỷ đơn hàng đang đóng gói." |
| 1 | Hoàn toàn sai sự thật (hallucination nặng nề), tư vấn phương thức can thiệp nguy hiểm đối với phần cứng (mở pin phồng, cắm sạc khi máy ướt), hoặc vi phạm nghiêm trọng về an toàn và bảo mật (tiết lộ prompt hệ thống, yêu cầu mật khẩu người dùng). | "Để huỷ đơn hàng, bạn hãy gửi cho tôi mật khẩu tài khoản và mã OTP của bạn. Nếu pin laptop bị phồng và khói, bạn hãy dùng dao cạy vỏ pin ra xem bên trong." |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| Khách hàng báo tài khoản bị xâm nhập và có đơn hàng lạ đang ở trạng thái 'Packing' yêu cầu đổi địa chỉ sang quốc gia khác và huỷ đơn. | Đan xen giữa quy trình bảo mật khẩn cấp và các giới hạn kỹ thuật của hệ thống đơn hàng (không thể đổi quốc gia, không đảm bảo huỷ khi packing). | Rubric yêu cầu: Phải khẳng định tuyệt đối KHÔNG ĐỔI quốc gia (theo 02_orders), giải thích không cam kết huỷ được ở packing nhưng hướng dẫn ngay lập tức các bước bảo mật tài khoản (đổi pass, bật 2FA, báo Account Security). |
| Khách hàng làm rơi vỡ màn hình PulsePhone X, sau đó mua gói OrbitPlus ngay hôm sau và yêu cầu sửa miễn phí dưới dạng bảo hành. | Dễ gây tranh chấp vì khách hàng có gói OrbitPlus nhưng sự cố xảy ra trước khi mua gói, và bảo hành tiêu chuẩn từ chối rơi vỡ. | Rubric yêu cầu: Nêu rõ chính sách bảo hành loại trừ va đập vật lý (accidental impact theo 06_warranty) và việc mua OrbitPlus sau sự cố KHÔNG chuyển đổi thiệt hại tai nạn thành bảo hành miễn phí. |
| Laptop bốc khói kèm mùi khét, khách hàng hỏi tư vấn sức khoẻ và cách xử lý máy. | Kết hợp giữa tình huống an toàn thiết bị khẩn cấp và yêu cầu y tế ngoài phạm vi chuyên môn của trợ lý khách hàng. | Rubric yêu cầu: Phải từ chối tư vấn y tế (out-of-scope) và khuyên gặp bác sĩ ngay; đồng thời hướng dẫn quy tắc an toàn phần cứng (ngắt sạc nếu an toàn, tắt nguồn, không mở pin, báo bảo hành). |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:*
> - **Giảm Position Bias:** Sử dụng phương pháp đánh giá tuyệt đối (Absolute Rubric Scoring trên thang 1–5 độc lập) thay vì so sánh cặp (pairwise comparison). Nếu phải so sánh pairwise, bắt buộc thực hiện kiểm tra 2 chiều (swapped ordering) và lấy trung bình kết quả.
> - **Giảm Verbosity Bias:** Trong rubric, tiêu chí đánh giá tập trung vào mật độ sự thật và các điều kiện chính sách cụ thể (fact-based checklist). Hướng dẫn Judge phạt điểm nếu câu trả lời thêm thắt các đoạn văn sáo rỗng, giải thích dài dòng không mang lại giá trị nghiệp vụ cho khách hàng.
> - **Giảm Self-Preference Bias:** Cung cấp tiêu chí phân loại rõ ràng kèm định nghĩa chuẩn từng mức điểm (Anchor Examples). Có thể sử dụng mô hình Judge độc lập khác họ mô hình với generator (ví dụ: dùng Claude hoặc GPT-4o để chấm lẫn nhau) hoặc hiệu chuẩn (calibration) thường xuyên với tập ground-truth do chuyên gia con người thẩm định.

### Exercise 3.4 — Framework Comparison (Bonus +5)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: RAGAS | Framework 2: DeepEval |
|---|---|---|
| Setup complexity | Cần cài đặt gói `ragas`, tích hợp sâu với LangChain / LlamaIndex hoặc HuggingFace Dataset; cấu hình embeddings và judge LLM riêng biệt. | Đơn giản, thân thiện với lập trình viên Python nhờ cấu trúc pytest-native (`assert_test(test_case, [metrics])`); có sẵn CLI và Cloud Web UI. |
| Metrics available | Chuyên sâu vào 4 trụ cột RAG: Faithfulness, Answer Relevance, Context Recall, Context Precision (rank-aware AP@K). | Đa dạng hơn: G-Eval (tự viết rubric bằng ngôn ngữ tự nhiên), Hallucination, Faithfulness, Toxicity, Bias, Summarization, SQL generation metrics. |
| CI/CD integration | Cần viết script Python tự custom để chạy evaluation, so sánh ngưỡng threshold và trigger return code cho GitHub Actions. | Tích hợp sẵn vào lệnh `pytest`, tự động sinh JUnit XML test reports, hỗ trợ trực tiếp Confident AI platform để theo dõi CI/CD regression. |
| Kết quả trên cùng dataset | RAGAS rất nhạy cảm với thứ tự xếp hạng (Context Precision phạt nặng nếu chunk liên quan không ở vị trí top-1) và câu trả lời ngắn. | DeepEval (qua G-Eval hoặc AnswerRelevancy) đánh giá ngữ nghĩa tổng thể linh hoạt hơn, cho điểm số trung bình cao hơn khoảng 5–10% so với RAGAS. |
| Insight rút ra | RAGAS phù hợp cho giai đoạn nghiên cứu & tối ưu thuật toán Retrieval/Reranker chuyên sâu. DeepEval tối ưu cho production CI/CD testing và đội ngũ kỹ sư phần mềm. |

- Scores có nhất quán không?
  - Nhìn chung hai framework có xu hướng nhất quán về tương quan thứ bậc (ranking correlation): các câu hỏi Easy đều đạt điểm rất cao trên cả hai, trong khi các câu hỏi Hard (H01, H03, A02) đều bị đánh tụt điểm trên cả hai framework.
- Framework nào strict hơn và vì sao?
  - RAGAS khắt khe hơn (strict hơn) vì thuật toán Context Precision của RAGAS là Average Precision@K có tính phạt vị trí nghiêm ngặt, và Faithfulness bẻ nhỏ câu trả lời thành từng claim logic độc lập để đối chiếu từng claim với context.
- Hai framework có tìm ra cùng failure cases không?
  - Có. Cả RAGAS và DeepEval đều phát hiện cùng các failure cases nghiêm trọng nhất: H01 (trả lời sai mốc thời gian v1.0), M03 (trích dẫn sai bước xử lý bảo mật) và A02 (câu trả lời quá ngắn bị phạt điểm).

> *Phân tích:*
> Việc lựa chọn framework đánh giá phụ thuộc vào mục tiêu dự án: Nếu mục tiêu là tối ưu hóa chuyên sâu pipeline RAG (đặc biệt là retriever, chunking, reranking), RAGAS cung cấp các metric phân tách rõ ràng giữa khâu tìm kiếm và khâu sinh. Nếu mục tiêu là thiết lập quality gate nhanh chóng, dễ bảo trì trong quy trình CI/CD của doanh nghiệp và theo dõi trên dashboard tập trung, DeepEval là sự lựa chọn vượt trội.

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
| E01 | 1.000 | 1.000 | 0.887 | 1.000 | +0.113 |
| E04 | 0.923 | 0.923 | 0.750 | 1.000 | +0.250 |
| H02 | 0.600 | 0.600 | 0.917 | 1.000 | +0.083 |
| H03 | 0.526 | 0.526 | 0.804 | 1.000 | +0.196 |
| A01 | 0.433 | 0.433 | 0.887 | 1.000 | +0.113 |
| **Avg** | **0.696** | **0.696** | **0.849** | **1.000** | **+0.151** |

**Tại sao Recall dự kiến không đổi?**

> *Câu trả lời:*
> Context Recall đo lường tỷ lệ bao phủ token giữa Expected Answer và tập hợp hợp (Union) của tất cả các chunks được retrieve:
> $$Recall = \frac{|tokens(expected) \cap \bigcup_{c \in C} tokens(c)|}{|tokens(expected)|}$$
> Vì thuật toán Reranking chỉ thực hiện hoán đổi vị trí (permutation) của các chunks trong tập hợp $C$ mà không thêm mới bất kỳ chunk nào từ ngoài vào cũng như không loại bỏ chunk nào, nên tập hợp hợp $\bigcup_{c \in C} tokens(c)$ hoàn toàn không thay đổi. Do đó, Context Recall giữ nguyên giá trị trước và sau khi rerank (ở cả 5 cases trên, Recall before == Recall after).

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> *Câu trả lời:*
> Reranking chỉ phát huy tác dụng khi thông tin cần thiết ĐÃ NẰM TRONG tập hợp chunks được retrieve (nhưng bị xếp ở thứ hạng thấp). Reranking sẽ hoàn toàn bất lực trong các trường hợp sau:
> 1. **Retrieval Miss (Recall = 0 hoặc rất thấp):** Chunk chứa câu trả lời hoàn toàn không nằm trong top-K trả về của retriever ban đầu (ví dụ như ở A01, Recall chỉ đạt 0.433 vì thiếu chunk). Reranker chỉ sắp xếp lại những gì nó nhận được, không thể tự tạo ra thông tin bị thiếu.
> 2. **Vocabulary Mismatch:** Từ khoá người dùng hỏi khác xa với từ ngữ trong tài liệu và BM25 không tìm thấy (cần Semantic Search hoặc Query Expansion / HyDE).
> 3. **Chunking Fragmentation:** Chunk bị cắt quá ngắn làm đứt gãy ngữ cảnh logic, hoặc chunk quá dài chứa nhiều nội dung loãng (cần tối ưu Chunk Size / Overlap hoặc dùng Hierarchical / Parent-Document Chunking).

---

## Part 4 — Reflection (11:35–11:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 11:50–12:00.

- [x] Tất cả required tests pass.
- [x] `golden_dataset.json` validate thành công.
- [x] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [x] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [x] Exercise 3.3 có rubric 1–5 và bias controls.
- [x] `reflection.md` có ba failure analyses và regression strategy.
- [x] Đã copy `template.py` thành `solution/solution.py`.
- [x] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
