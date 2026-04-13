# Báo Cáo Cá Nhân — Lab Day 08: RAG Pipeline

**Họ và tên:** Trần Đình Minh Vương  
**Vai trò trong nhóm:** Eval & Docs Owner  
**Ngày nộp:** 2026-04-13  
**Độ dài:** 680 từ

---

## 1. Tôi đã làm gì trong lab này?

Trong lab này, tôi chịu trách nhiệm Sprint 4 (Evaluation) và documentation. Công việc chính:

Đầu tiên, implement evaluation pipeline trong `eval.py` với 4 scoring functions sử dụng LLM-as-Judge (GPT-4o-mini). Mỗi metric có logic riêng: Context Recall tính recall từ expected sources, Faithfulness dùng LLM kiểm tra grounding, Relevance đánh giá answer có trả lời đúng câu hỏi, Completeness so sánh với expected answer.

Thứ hai, implement `run_scorecard()` để chạy tự động 10 test questions, chấm điểm và generate markdown report. Function tích hợp với `rag_answer()` của team, nhận results và chấm theo 4 metrics.

Thứ ba, implement `compare_ab()` để so sánh baseline vs variant, tính delta và export CSV. Kết quả: Baseline tốt hơn Variant ở 3/4 metrics.

Cuối cùng, viết documentation cho `docs/architecture.md` và `reports/group_report.md` với phân tích challenges và recommendations.

---

## 2. Điều tôi hiểu rõ hơn sau lab này

Sau lab này, tôi hiểu sâu hơn về evaluation trong RAG systems. RAG có nhiều failure modes - retrieval lấy sai docs, generation hallucinate, hoặc answer đúng nhưng thiếu chi tiết. Vì vậy cần nhiều metrics để đo các aspects khác nhau, không chỉ "đúng/sai" đơn giản.

Tôi cũng hiểu về LLM-as-Judge - technique mạnh để automate evaluation. Học được cách design prompt cho judge: rõ ràng về scale, yêu cầu structured JSON output, provide examples. GPT-4o-mini chấm khá consistent nhưng đôi khi strict quá (q03 faithfulness = 2 có thể false negative).

Quan trọng nhất: "more complex ≠ better". Evaluation cho thấy Baseline (Dense) tốt hơn Variant (Hybrid) với faithfulness 4.60 vs 4.10. Phải test và measure impact thực tế, không assume advanced technique luôn tốt hơn. A/B testing đúng cách giúp identify root cause: BM25 match keyword quá aggressive.

---

## 3. Điều tôi ngạc nhiên hoặc gặp khó khăn

Điều ngạc nhiên nhất là kết quả A/B comparison. Tôi kỳ vọng Hybrid ngang bằng Dense, nhưng thực tế kém hơn đáng kể: Faithfulness giảm 0.50 điểm, Completeness giảm 0.40 điểm. Phải deep dive per-question analysis để hiểu tại sao.

Khó khăn lớn nhất là debug q06 về "Escalation P1". Baseline trả lời đúng (completeness 5/5), Variant trả lời về access control (completeness 2/5) - sai focus hoàn toàn. Trace thấy BM25 match "P1" trong access control docs, đẩy chunks không relevant lên top.

Khó khăn khác là thiết kế Context Recall scoring. Ban đầu exact matching fail vì format khác nhau ("policy/refund-v4.pdf" vs "policy_refund_v4.txt"). Phải chuyển sang partial matching bằng filename lowercase mới work.

Cũng gặp LLM-as-Judge timeout khi chấm 10 câu liên tục. Thêm error handling và retry logic để ổn định.

---

## 4. Phân tích một câu hỏi trong scorecard

**Câu hỏi:** q06 - "Escalation trong sự cố P1 diễn ra như thế nào?"

**Phân tích:**

Câu này cho thấy rõ tại sao Hybrid kém hơn Dense. Expected: "Ticket P1 auto-escalate lên Senior Engineer nếu không phản hồi trong 10 phút".

**Baseline:** Faithfulness 5/5, Completeness 5/5. Trả lời chính xác: "Nếu không có phản hồi trong 10 phút, tự động escalate lên Senior Engineer." Dense retrieval lấy đúng chunks từ SLA document.

**Variant:** Faithfulness 5/5, Completeness 2/5. Trả lời về "cấp quyền tạm thời trong P1" - sai focus hoàn toàn. Nói về access control thay vì escalation.

**Root cause:** BM25 match "P1" quá aggressive. Từ "P1" xuất hiện trong cả SLA và access control docs. BM25 đẩy access control chunks lên top vì nhiều exact matches, mất chunks về escalation. Đây là keyword frequency problem - khi term xuất hiện nhiều, BM25 không phân biệt context nào relevant.

**Lesson:** Lỗi ở retrieval, không phải generation. Model generate đúng dựa trên chunks nhận được, nhưng chunks sai từ đầu. "Garbage in, garbage out".

---

## 5. Nếu có thêm thời gian, tôi sẽ làm gì?

Nếu có thêm 2-3 giờ, tôi sẽ implement ensemble judging - chạy 3 judges (GPT-4o-mini, GPT-4, Claude) và lấy median score. Kết quả cho thấy q03 faithfulness = 2 có thể false negative, ensemble giảm variance.

Tôi cũng sẽ thêm more test questions cho edge cases: multi-hop questions (combine info từ 2+ docs), numerical questions (SLA time, refund days), negation questions ("Sản phẩm nào KHÔNG được hoàn tiền?"). Hiện tại 10 câu chưa đủ test thoroughly.

Cuối cùng, implement automated regression testing. Mỗi khi tune pipeline, chạy lại scorecard và compare với previous results. Track improvement over time thay vì chỉ evaluate một lần cuối.
