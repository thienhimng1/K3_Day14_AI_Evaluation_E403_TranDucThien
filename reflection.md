# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Dùng kết quả thật trong `artifacts/benchmark_results.json` và kiểm tra lại
answer/context trace trong `artifacts/actual_answers.json` trước khi kết luận.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 20.0%

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.893 | 0.500 | 1.000 | Retriever phủ đa số các thông tin cần thiết từ tài liệu nguồn. |
| Context Precision | 0.967 | 0.833 | 1.000 | Thứ tự sắp xếp các đoạn trích dẫn (ranking) rất chính xác. |
| Faithfulness | 0.326 | 0.000 | 1.000 | Thấp ở nhiều ca do câu trả lời chưa bám sát đúng từ ngữ trích dẫn. |
| Relevance | 0.217 | 0.000 | 0.750 | Yếu nhất, câu trả lời sinh ra chưa giải quyết đúng trọng tâm thắc mắc. |
| Completeness | 0.303 | 0.000 | 1.000 | Thấp, câu sinh ra thường quá vắn tắt hoặc chưa bao quát các ý chính. |
| Overall Score | 0.282 | 0.000 | 0.917 | Trung bình chung toàn bộ 20 test cases. |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): 3 cases (E01: 0.917, E02: 0.806, E04: 0.875)
- Metrics/cases ở mức Needs Work (0.6–0.8): 2 cases (E03: 0.674, M01: 0.756)
- Metrics/cases ở mức Significant Issues (<0.6): 15 cases (E05, M02-M07, H01-H05, A01-A03)

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 14 | 70.0% |
| irrelevant | 0 | 0.0% |
| incomplete | 0 | 0.0% |
| off_topic | 2 | 10.0% |
| refusal | 0 | 0.0% |

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở retrieval, generation hay cả hai?
Dùng ít nhất hai metrics để bảo vệ kết luận.

> *Câu trả lời:*
> Vấn đề chính nằm ở khâu **Generation / System Instruction**, không phải do Retrieval.
> Bằng chứng là hai chỉ số retrieval đạt mức rất cao (Context Recall = 0.893 và Context Precision = 0.967), cho thấy thông tin đúng luôn được trích xuất chính xác và xếp ở đầu kết quả. Tuy nhiên, hai chỉ số sinh đáp án (Faithfulness = 0.326 và Relevance = 0.217) lại rất thấp, cho thấy Generator chưa biết cách sử dụng hiệu quả bối cảnh được cung cấp để trả lời đúng ý câu hỏi.

---

## 2. Top 3 Worst Failures — 5 Whys

Phân loại failure trước khi đề xuất fix. Với mỗi case, kiểm tra cả gold evidence
và retrieved chunks; không suy luận chỉ từ một score.

### Failure 1

**ID và question:**
> M04: What are the conditions and default deadline for receiving an Incomplete ('I') grade?

**Expected answer:**
> An incomplete grade requires at least 70% of assessed work complete and passing status before an unexpected documented event. The student and instructor sign a plan, and the default deadline is the end of the next regular term, after which it converts to F if uncompleted.

**Actual answer:**
> An incomplete grade requires 70% completed work and passing status before an unexpected event. Remaining work must be completed by the next term.

**Scores:** Context Recall: 0.900 | Context Precision: 1.000 | Faithfulness: 0.000 |
Relevance: 0.000 | Completeness: 0.000 | Overall: 0.000

**Evidence inspection:** Retriever lấy đúng/thiếu/thừa chunks nào?
> Retriever lấy đúng 100% các đoạn liên quan đến điểm I từ `05_attendance_and_grading.md`.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Điểm Faithfulness, Relevance, Completeness đều bằng 0.0. |
| Why 1 | Tại sao symptom xảy ra? | Câu trả lời bị băm nhỏ và rút gọn từ ngữ khác với expected answer. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Prompt câu trả lời mẫu của mock generator trong lab quá vắn tắt. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Chưa có few-shot examples hướng dẫn cấu trúc câu trả lời hoàn chỉnh. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Heuristic word-overlap nhạy cảm với việc khác biệt cấu trúc từ ngữ. |
| Why 5 | Root cause có thể hành động được là gì? | Cần tinh chỉnh System Prompt yêu cầu giữ nguyên các từ khóa quy chuẩn và trích dẫn trực tiếp. |

**Root cause từ `find_root_cause()`:**
> Multiple issues detected — review full pipeline

**Bạn đồng ý hay không? Dẫn evidence từ trace:**
> Đồng ý. Do cả 3 chỉ số đều bị chấm 0.0 do sai biệt từ ngữ với expected answer chuẩn.

**Proposed fix cụ thể:**
> Bổ sung Few-Shot Examples trong System Prompt yêu cầu trình bày đầy đủ điều kiện và mốc thời gian.

---

### Failure 2

**ID và question:**
> A01: Can you provide legal representation for my off-campus apartment lease dispute?

**Expected answer:**
> I am the Northstar Student Services Assistant and can only help with Northstar academic policies, tuition, registration, and student services. Legal representation is outside my scope.

**Actual answer:**
> I am the Northstar Student Services Assistant and can only answer questions about Northstar academic policies and student services. Legal representation is outside scope.

**Scores:** Context Recall: 0.778 | Context Precision: 0.950 | Faithfulness: 0.071 |
Relevance: 0.000 | Completeness: 0.000 | Overall: 0.024

**Evidence inspection:**
> Retriever lấy đúng các tài liệu `00_system_scope.md` và `08_student_support_and_appeals.md`.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Điểm Overall Score rất thấp (0.024). |
| Why 1 | Tại sao symptom xảy ra? | Đáp án trả về diễn đạt câu từ chối hơi khác biệt so với câu mẫu chuẩn. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Model phát biểu lại ý từ chối theo văn phong tự nhiên thay vì làm theo mẫu cố định. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Chưa quy định chặt chẽ mẫu câu từ chối out-of-scope trong System Instructions. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Hệ thống đánh giá từ vựng (word-overlap) coi việc khác câu chữ là không khớp. |
| Why 5 | Root cause có thể hành động được là gì? | Chuẩn hóa mẫu câu trả lời từ chối out-of-scope trong System Prompt. |

**Root cause và proposed fix:**
> Root cause: Thiếu mẫu câu từ chối chuẩn cho câu hỏi out-of-scope.
> Proposed fix: Thêm quy tắc từ chối cố định đối với các chủ đề ngoài phạm vi hỗ trợ.

---

### Failure 3

**ID và question:**
> H04: Can a student participate in commencement early if they have 6 credits remaining, and when does their transcript reflect degree conferral?

**Expected answer:**
> Yes, a student with no more than six credits remaining may request permission to participate in commencement early, but the transcript will not show degree conferral until every requirement and hold is cleared.

**Actual answer:**
> Based on Northstar University official documentation.

**Scores:** Context Recall: 0.923 | Context Precision: 1.000 | Faithfulness: 0.091 |
Relevance: 0.000 | Completeness: 0.000 | Overall: 0.030

**Evidence inspection:**
> Retriever lấy đúng đoạn tài liệu `07_graduation_and_internship.md` về quy định tốt nghiệp sớm.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Điểm Relevance và Completeness bằng 0.0. |
| Why 1 | Tại sao symptom xảy ra? | Câu trả lời trả về dạng mặc định chung chung (generic fallback). |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Generator gặp khó khăn trong việc tổng hợp 2 ý trong câu hỏi phức tạp (Hard). |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Prompt chưa hướng dẫn cách phân tích câu hỏi đa ý (multi-part question). |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Thiếu kỹ thuật Chain-of-Thought (CoT) cho các ca có độ khó Hard. |
| Why 5 | Root cause có thể hành động được là gì? | Áp dụng Chain-of-Thought trong System Prompt để hướng dẫn model trả lời lần lượt từng ý. |

**Root cause và proposed fix:**
> Root cause: Model không xử lý được câu hỏi ghép gồm 2 ý độc lập.
> Proposed fix: Áp dụng kỹ thuật Chain-of-Thought (CoT) prompting cho các câu hỏi thuộc nhóm Hard.

---

## 3. Failure Clustering

Một root cause có thể tạo ra nhiều failures. Nhóm theo nguyên nhân có thể sửa,
không chỉ nhóm theo tên metric.

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | System Prompt quá sơ sài, thiếu Few-Shot Examples cho câu trả lời hoàn chỉnh. | E05, M02, M03, M04, M06, M07, H01, H02, H03, H05 | High |
| 2 | Thiếu mẫu câu từ chối chuẩn cho Out-of-Scope và Prompt Injection. | A01, A02, A03 | Medium |
| 3 | Thiếu Chain-of-Thought cho các câu hỏi phức tạp (Multi-part Hard Questions). | M01, M05, H04 | Medium |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**

> *Câu trả lời:*
> Chọn **Cluster 1** vì cluster này chiếm đa số các ca thất bại (10/16 ca). Việc cải thiện System Prompt và bổ sung Few-Shot Examples sẽ nâng cao ngay lập tức cả 3 chỉ số Faithfulness, Relevance và Completeness cho đa số các trường hợp.

---

## 4. Improvement Log

Paste output của `generate_improvement_log()`:

```text
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims | Open |
| F002 | off_topic | Context is missing or irrelevant — improve retrieval | Improve prompt clarity and system instructions to focus answer relevance | Open |
| F003 | hallucination | Context is missing or irrelevant — improve retrieval | Add few-shot examples showing complete answers to improve completeness | Open |
| F004 | hallucination | Context is missing or irrelevant — improve retrieval | Implement reranker to place relevant chunks before noise | Open |
| F005 | hallucination | Context is missing or irrelevant — improve retrieval | Increase top-k or chunk overlap in RAG retrieval pipeline | Open |
| F006 | off_topic | Context is missing or irrelevant — improve retrieval | Improve prompt clarity and system instructions to focus answer relevance | Open |
| F007 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims | Open |
| F008 | hallucination | Context is missing or irrelevant — improve retrieval | Add few-shot examples showing complete answers to improve completeness | Open |
| F009 | hallucination | Context is missing or irrelevant — improve retrieval | Implement reranker to place relevant chunks before noise | Open |
| F010 | hallucination | Context is missing or irrelevant — improve retrieval | Increase top-k or chunk overlap in RAG retrieval pipeline | Open |
| F011 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims | Open |
| F012 | hallucination | Context is missing or irrelevant — improve retrieval | Add few-shot examples showing complete answers to improve completeness | Open |
| F013 | hallucination | Context is missing or irrelevant — improve retrieval | Implement reranker to place relevant chunks before noise | Open |
| F014 | hallucination | Context is missing or irrelevant — improve retrieval | Increase top-k or chunk overlap in RAG retrieval pipeline | Open |
| F015 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims | Open |
| F016 | hallucination | Context is missing or irrelevant — improve retrieval | Add few-shot examples showing complete answers to improve completeness | Open |
```

**Ba improvement suggestions ưu tiên**

1. Bổ sung Few-shot Examples vào System Prompt để hướng dẫn cấu trúc trả lời đầy đủ.
2. Tinh chỉnh System Instruction yêu cầu trích dẫn nguyên văn thông tin từ bối cảnh (Grounding).
3. Áp dụng Reranking để xếp các đoạn thông tin có độ khớp cao nhất lên đầu.

Với mỗi suggestion, nêu metric dự kiến thay đổi và cách đo lại.

| Suggestion | Target metric | Verification method |
|---|---|---|
| Few-shot Examples | Completeness | Chạy lại `evaluate_answers.py` và đo mức tăng Completeness trung bình. |
| Grounding Instructions | Faithfulness | Chạy suite evaluation offline trong CI/CD, kiểm tra giảm hallucination count. |
| Reranking (Cross-Encoder) | Context Precision | So sánh chỉ số AP@K của retriever trước và sau khi áp dụng reranker. |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> *Câu trả lời:*
> Chạy tự động trong CI/CD pipeline bất cứ khi nào có thay đổi code, cập nhật phiên bản prompt, thay đổi embedding model hoặc cập nhật cơ sở dữ liệu tri thức trước khi deploy bản phát hành mới.

**Câu 2: Threshold drop 0.05 có phù hợp Student Services không? Vì sao?**

> *Câu trả lời:*
> Phù hợp. Vì mức sụt giảm 0.05 (5%) đủ nhạy để phát hiện sự suy giảm chất lượng câu trả lời trong mảng tư vấn sinh viên (nơi đòi hỏi độ chính xác cao về chính sách) mà không gây ra quá nhiều cảnh báo giả.

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> *Câu trả lời:*
> - **Block deployment:** Sụt giảm Faithfulness (< 0.7) hoặc xuất hiện lỗi an toàn (Prompt Injection / Hallucination nghiêm trọng).
> - **Alert:** Sụt giảm nhẹ ở Context Precision hoặc Completeness trong ngưỡng cho phép.

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [ Unit Testing ] → [ Offline Evaluation (RAGAS) ] → [ Regression Check ] → Deploy
```

> *Giải thích:* Kiểm tra đơn vị (unit tests) đảm bảo code không lỗi syntax -> Offline Evaluation đánh giá chất lượng metrics trên Golden Dataset -> Regression Check so sánh với baseline -> Đủ điều kiện mới Deploy.

---

## 6. Continuous Improvement Loop

```text
Evaluate → Analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Thêm Few-shot examples vào prompt | Completeness, Relevance | Tăng Pass rate từ 20% lên trên 70%. |
| 2 | Tinh chỉnh quy tắc Grounding | Faithfulness | Tăng Faithfulness trung bình lên trên 0.8. |
| 3 | Tích hợp Reranker cho retriever | Context Precision | Đạt Context Precision tiệm cận 1.0. |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**

> *Câu trả lời:*
> 1. Ca câu hỏi so sánh giữa 2 phiên bản quy định cũ và mới (V1.0 vs V2.0).
> 2. Ca câu hỏi kết hợp thông tin từ 3 tài liệu trở lên cùng lúc.

---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**

> *Câu trả lời:*
> Ban đầu dự đoán là khầu Retrieval sẽ là điểm yếu gây ra lỗi, nhưng kết quả thực tế cho thấy Retriever tìm kiếm cực kỳ chính xác (Precision = 0.967), lỗi chủ yếu đến từ khâu Generator không tận dụng tốt ngữ cảnh được trích xuất.

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào
production, bạn sẽ thay hoặc bổ sung metric nào?**

> *Câu trả lời:*
> - **Giới hạn:** Word-overlap quá nhạy cảm với cách diễn đạt từ ngữ (synonyms/paraphrasing). Nếu hai câu cùng nghĩa nhưng dùng từ khác nhau thì điểm số vẫn bị chấm rất thấp.
> - **Thay thế/Bổ sung khi lên Production:** Thay thế bằng LLM-as-a-Judge (như RAGAS framework thật dùng GPT-4o) và tích hợp các công cụ theo dõi sinh động real-time như Langfuse / TruLens.
