# Grading Questions Report

Ngày chạy: 2026-04-13
Nguồn câu hỏi: data/grading_questions.json
Số câu: 10

## 1. Cấu hình đã test

### Baseline
- retrieval_mode: dense
- top_k_search: 10
- top_k_select: 3
- use_rerank: False

### Variant
- retrieval_mode: hybrid
- top_k_search: 10
- top_k_select: 3
- use_rerank: True

## 2. Kết quả tổng quan

| Metric | Baseline (Dense) | Variant (Hybrid + Rerank) | Delta (Variant - Baseline) |
|---|---:|---:|---:|
| Faithfulness | 4.30 | 4.20 | -0.10 |
| Relevance | 3.40 | 3.40 | 0.00 |
| Context Recall | 5.00 | 5.00 | 0.00 |
| Completeness | 2.70 | 2.60 | -0.10 |

Kết luận nhanh:
- Variant hiện tại chưa vượt baseline trên bộ grading_questions.
- Retrieval recall rất cao ở cả 2 cấu hình, nhưng completeness còn thấp.

## 3. Kết quả theo từng câu hỏi

| ID | Baseline (F/R/CR/C) | Variant (F/R/CR/C) | Nhận xét nhanh |
|---|---|---|---|
| gq01 | 5 / 4 / 5 / 2 | 4 / 4 / 5 / 2 | Baseline faithfulness nhỉnh hơn |
| gq02 | 5 / 4 / 5 / 3 | 5 / 4 / 5 / 3 | Tương đương |
| gq03 | 5 / 5 / 5 / 4 | 5 / 5 / 5 / 4 | Tương đương, tốt |
| gq04 | 4 / 3 / 5 / 3 | 4 / 3 / 5 / 2 | Variant thiếu ý hơn |
| gq05 | 2 / 2 / 5 / 1 | 2 / 2 / 5 / 1 | Điểm thấp ở cả hai |
| gq06 | 5 / 2 / 5 / 4 | 5 / 2 / 5 / 4 | Relevance thấp ở cả hai |
| gq07 | 3 / 2 / N/A / 1 | 3 / 2 / N/A / 1 | Câu thiếu context, cần abstain chuẩn |
| gq08 | 4 / 4 / 5 / 3 | 4 / 4 / 5 / 3 | Tương đương |
| gq09 | 5 / 3 / 5 / 2 | 5 / 3 / 5 / 2 | Tương đương |
| gq10 | 5 / 5 / 5 / 4 | 5 / 5 / 5 / 4 | Tương đương, tốt |

Ghi chú cột điểm:
- F: Faithfulness
- R: Relevance
- CR: Context Recall
- C: Completeness

## 4. Điểm cần cải thiện trước demo/chấm

1. Tăng Completeness cho câu hỏi nhiều điều kiện (gq05, gq07).
2. Cải thiện Relevance cho câu hỏi đa ý (gq06).
3. Siết logic abstain/out-of-scope để không trả lời thiếu căn cứ.
4. Thử lại variant theo nguyên tắc đổi 1 biến/lần:
   - Hybrid không rerank
   - Dense + rerank
   - Hybrid + ngưỡng chọn chunk khác

## 5. Tệp liên quan

- Bộ câu hỏi: data/grading_questions.json
- Kết quả baseline hiện có: results/scorecard_baseline.md
- Script chấm: eval.py
- Pipeline trả lời: rag_answer.py
