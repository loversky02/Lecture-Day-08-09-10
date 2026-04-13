# Group Report — Lab Day 08: RAG Pipeline

**Nhóm:** [Tên nhóm]  
**Thành viên:**
- [Tên 1] - Tech Lead
- [Tên 2] - Retrieval Owner
- [Tên 3] - Eval Owner
- [Tên 4] - Documentation Owner

**Ngày nộp:** 2026-04-13

---

## 1. Tổng quan hệ thống

Nhóm đã xây dựng RAG pipeline hoàn chỉnh với 4 sprints:
- Sprint 1: Indexing 29 chunks từ 5 tài liệu
- Sprint 2: Baseline với Dense retrieval + Grounded prompt
- Sprint 3: Variant với Hybrid retrieval (Dense + BM25 + RRF)
- Sprint 4: Evaluation với LLM-as-Judge

## 2. Quyết định kỹ thuật chính

### Chunking Strategy
- **Quyết định:** Section-based chunking theo heading "=== ... ==="
- **Lý do:** Giữ nguyên cấu trúc tự nhiên của tài liệu, không cắt giữa điều khoản
- **Tham số:** 400 tokens chunk size, 80 tokens overlap
- **Kết quả:** 29 chunks với metadata đầy đủ, không có chunk bị cắt giữa section

### Embedding Model
- **Quyết định:** Sentence Transformers (paraphrase-multilingual-MiniLM-L12-v2) local
- **Lý do:** Miễn phí, nhanh, không cần API key, đủ tốt cho tiếng Việt
- **Trade-off:** Chất lượng embedding thấp hơn OpenAI một chút nhưng tiết kiệm chi phí

### Retrieval Strategy
- **Baseline:** Dense (embedding similarity)
- **Variant:** Hybrid (Dense + BM25 + RRF với weights 0.6/0.4)
- **Lý do chọn Hybrid:** Corpus có cả natural language và keyword/mã lỗi
- **Kết quả:** Baseline tốt hơn Variant vì BM25 quá aggressive với keyword phổ biến

### Grounded Prompt
- **4 quy tắc:** Evidence-only, Abstain, Citation, Short/clear
- **Kết quả:** Faithfulness 4.60/5, abstain đúng 100% với câu không có docs

## 3. Kết quả Evaluation

### Metrics (Kết quả thực tế từ LLM-as-Judge)
| Metric | Baseline (Dense) | Variant (Hybrid) | Delta |
|--------|------------------|------------------|-------|
| Faithfulness | 4.60/5 | 4.10/5 | **-0.50** ⚠️ |
| Answer Relevance | 4.40/5 | 4.30/5 | -0.10 |
| Context Recall | 5.00/5 | 5.00/5 | 0.00 ✅ |
| Completeness | 4.60/5 | 4.20/5 | **-0.40** ⚠️ |

### Phân tích chi tiết

**Điểm mạnh của Baseline:**
- Context Recall hoàn hảo (5.00/5) - retrieve đúng 100% expected sources
- Faithfulness cao (4.60/5) - answer bám sát evidence
- Completeness tốt (4.60/5) - đầy đủ thông tin quan trọng
- Abstain đúng 100% với câu không có docs (q09)

**Vấn đề của Variant (Hybrid):**
- Faithfulness giảm mạnh (-0.50) do BM25 retrieve chunks không relevant
- Completeness giảm (-0.40) vì chunks sai focus
- Ví dụ q06: Hybrid retrieve về "access control" thay vì "SLA escalation" vì keyword "P1" match quá nhiều

**Câu hỏi Variant kém hơn:**
- q05: Faithfulness 5→4 (thêm thông tin không chắc chắn)
- q06: Completeness 5→2 (trả lời sai focus hoàn toàn)
- q09: Faithfulness 5→1 (abstain quá ngắn, thiếu context)

**Root Cause:**
BM25 trong Hybrid quá aggressive với keyword frequency cao (P1, Level 3, remote xuất hiện nhiều lần trong corpus), đẩy chunks ít semantic relevant lên top.

**Kết luận:** Dense retrieval đơn giản nhưng hiệu quả hơn cho corpus này. Hybrid cần fine-tuning BM25 weights hoặc better tokenization cho tiếng Việt.

## 4. Challenges và Solutions

### Challenge 1: Hybrid không cải thiện như kỳ vọng
- **Vấn đề:** Kỳ vọng Hybrid (Dense + BM25) tốt hơn Dense nhưng kết quả ngược lại
- **Root cause phát hiện:** 
  - BM25 match keyword quá aggressive với terms có frequency cao (P1, Level 3, remote)
  - Ví dụ q06: Query "Escalation P1" → BM25 match "P1" trong access control docs → retrieve sai context
  - Dense semantic search tốt hơn cho corpus có natural language nhiều
- **Lesson learned:** Phải test và hiểu đặc điểm corpus trước khi áp dụng advanced techniques. "More complex" không luôn = "better"
- **Potential fix:** Giảm BM25 weight xuống 0.2, hoặc implement better Vietnamese tokenization

### Challenge 2: Completeness có thể cải thiện
- **Vấn đề:** Completeness 4.60/5, một số câu thiếu chi tiết phụ (q07, q10)
- **Root cause:** Top-3 chunks đôi khi không cover hết thông tin liên quan
- **Evidence:** q07 thiếu mention về tên cũ "Approval Matrix", q10 thiếu clarify về VIP policy
- **Potential fix:** 
  - Tăng top_k_select lên 5 chunks
  - Implement cross-encoder rerank để chọn chunks tốt hơn
  - Thêm query expansion cho alias terms

### Challenge 3: Alias queries và semantic gap
- **Vấn đề:** q07 "Approval Matrix" không match tốt với "Access Control SOP" (completeness 4/5)
- **Root cause:** Embedding model không capture được relationship giữa alias terms
- **Potential fix:** 
  - Thêm metadata field "aliases" vào chunks
  - Query expansion: "Approval Matrix" → ["Approval Matrix", "Access Control", "SOP"]
  - Fine-tune embedding model với domain-specific synonyms

### Challenge 4: LLM-as-Judge consistency
- **Vấn đề:** q03 có faithfulness score = 2 vì LLM judge cho rằng "IT Security" không có trong context, nhưng thực tế có
- **Root cause:** GPT-4o-mini đôi khi strict quá mức hoặc miss details trong long context
- **Lesson:** LLM-as-Judge cần validation, không thể 100% tin tưởng
- **Potential fix:** Sử dụng GPT-4 (stronger model) hoặc ensemble multiple judges

## 5. Nếu có thêm thời gian

1. **Query Expansion:** Generate alternative phrasings cho alias queries
2. **Better Tokenization:** Word segmentation cho tiếng Việt trong BM25
3. **Cross-encoder Rerank:** Cải thiện top-3 selection
4. **More Test Questions:** Edge cases và multi-hop questions
5. **Production Considerations:** Error handling, logging, monitoring

## 6. Phân công công việc

| Thành viên | Vai trò | Công việc chính |
|-----------|---------|-----------------|
| [Tên 1] | Tech Lead | Sprint 1+2, pipeline integration |
| [Tên 2] | Retrieval Owner | Sprint 3, hybrid implementation |
| [Tên 3] | Eval Owner | Sprint 4, LLM-as-Judge |
| [Tên 4] | Documentation Owner | architecture.md, tuning-log.md |

## 7. Kết luận

Nhóm đã hoàn thành đầy đủ 4 sprints với RAG pipeline hoạt động tốt. Baseline (Dense) đạt faithfulness 4.60/5 và context recall 5.00/5. Variant (Hybrid) không cải thiện nhưng nhóm đã học được bài học quan trọng về việc hiểu đặc điểm data trước khi chọn advanced techniques.

Hệ thống sẵn sàng cho grading questions và có thể scale cho production với một số cải tiến về error handling và monitoring.
