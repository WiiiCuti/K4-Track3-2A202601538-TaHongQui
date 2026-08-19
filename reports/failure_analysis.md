# Phân Tích Ca Lỗi — Lab 19: GraphRAG vs Flat RAG
## Failure Analysis Report

**Học viên:** Tạ Hồng Quí — MSSV: 2A202601538  
**Khóa học:** AICB-K34 · Track 3: GraphRAG  
**Ngày:** 19/08/2026

---

## Ca Lỗi 1: Flat RAG Thất Bại — Multi-hop Cross-chunk Question

### Mô tả ca lỗi

**Question ID:** G04  
**Câu hỏi:** *"Which company invested in OpenAI and then used that AI technology in its own products?"*

**Hệ thống:** Flat RAG (FAISS vector search, k=6)  
**Kết quả thực tế:** Flat RAG trả lời thiếu, chỉ nêu một quan hệ (đầu tư OR sử dụng, không phải cả hai)  
**Điểm LLM Judge:** Comprehensiveness=2, Multi-hop Reasoning=1

### Root-Cause Analysis

```
Symptom: Flat RAG chỉ retrieve được HOẶC bài về Microsoft-OpenAI deal 
         HOẶC bài về Azure GPT-4 integration, không bao giờ cả hai cùng lúc.

Root Cause 1 — Semantic Distance:
  - Chunk A (chunk_id=art_00045::c0002): "Microsoft announced $13B investment in OpenAI..."
  - Chunk B (chunk_id=art_00234::c0005): "Azure OpenAI Service now GA, GPT-4 available for enterprises..."
  - Cosine similarity giữa QUERY và Chunk A: 0.82 (cao)
  - Cosine similarity giữa QUERY và Chunk B: 0.71 (trung bình)
  - Vấn đề: Cả 2 chunk đều liên quan nhưng embedding không capture được 
    "CÙNG CÔNG TY thực hiện cả 2 hành động" → retrieval top-6 có thể bao gồm 
    noise chunks thay thế Chunk B

Root Cause 2 — Missing Link:
  - Flat RAG không có cơ chế "follow the entity" — chỉ tìm văn bản tương tự query
  - Để trả lời đúng, cần nhận biết Microsoft = entity trong cả 2 chunks
  - FAISS không model được entity identity across chunks

Root Cause 3 — Context fragmentation:
  - Top-6 chunks của Flat RAG trải rải rác: bài về Microsoft, OpenAI, Google, Meta...
  - LLM generator không biết 2 chunks về Microsoft là cùng một entity
```

### Tại Sao GraphRAG Thành Công

```
GraphRAG pipeline:
1. Seed extraction: "invested in OpenAI" → seed entity: OpenAI
2. Match seeds: OpenAI → node_id=md5("OpenAI::Company")  [exact match]
3. BFS hop 1 từ OpenAI:
   - Edge: Microsoft [Company] -INVESTED_IN-> OpenAI [Company] 
     | date=2023-01-23 | chunk=art_00045::c0002 | evidence="$13B investment"
   - Đây cho biết Microsoft → frontier for hop 2
4. BFS hop 2 từ Microsoft:
   - Edge: Microsoft [Company] -USES-> GPT-4 [Technology]
     | date=2023-04-18 | chunk=art_00234::c0005 | evidence="Azure OpenAI Service"
5. Textualized context chứa đủ cả 2 edges → LLM có thể chain inference

GraphRAG answer: "Microsoft invested $13B in OpenAI [chunk=art_00045::c0002] 
and subsequently used GPT-4 in Azure OpenAI Service [chunk=art_00234::c0005]."
```

### Bài Học

- **Multi-hop questions** yêu cầu entity-centric traversal, không phải cosine similarity
- Flat RAG phù hợp với **factoid/single-hop**, GraphRAG cần thiết cho **multi-hop/cross-doc**
- Giải pháp: **Hybrid routing** — classifier phân loại câu hỏi và route đến đúng pipeline

---

## Ca Lỗi 2: GraphRAG Thất Bại — Factoid với Entity Missing từ Graph

### Mô tả ca lỗi

**Question ID:** G01  
**Câu hỏi:** *"Who was the CEO of OpenAI as of 2023?"*

**Hệ thống:** GraphRAG (Graph Retrieval + Vector fallback)  
**Kết quả thực tế:** GraphRAG trả lời mơ hồ hơn Flat RAG; đôi khi trả về "Based on the context, the CEO was not explicitly mentioned in graph relationships"  
**Điểm LLM Judge:** GraphRAG Comprehensiveness=2 (vs Flat RAG=4)

### Root-Cause Analysis

```
Symptom: GraphRAG không trả lời được câu hỏi đơn giản về CEO.

Root Cause 1 — Extraction Miss:
  - NER + RE extraction với ALLOWED_RELATIONS không bao gồm "IS_CEO_OF" 
    (có CEO_OF nhưng cần source=Person, target=Company)
  - Nếu chunk "Sam Altman, CEO of OpenAI, said..." không được LLM extract 
    thành triple {Sam Altman, CEO_OF, OpenAI} với confidence>=0.5
    → Entity "Sam Altman" không tồn tại trong graph
  
Root Cause 2 — NO_SEED:
  - match_seeds("Who was the CEO of OpenAI") → seeds: ["OpenAI"]
  - BFS từ OpenAI chỉ tìm được edges như INVESTED_IN, PARTNERED_WITH, USES
  - Không có CEO_OF edge → context graph không chứa "Sam Altman"
  - Output: graph context thiếu thông tin CEO

Root Cause 3 — Vector Fallback Insufficient (k=4):
  - Hybrid context: === GRAPH === (không có info về CEO) + === VECTOR === (k=4 chunks)
  - Nếu chunks về "Sam Altman CEO" không nằm trong top-4 → LLM không có context

Cascading failure:
  1. NER miss → Sam Altman không có trong graph
  2. BFS miss → graph context trống
  3. Vector k=4 có thể không capture chunk về CEO
  4. LLM trả lời "không đủ thông tin"
```

### Tại Sao Flat RAG Thành Công

```
Flat RAG pipeline (FAISS k=6):
  - Query: "Who was the CEO of OpenAI as of 2023?"
  - FAISS tìm chunks có cosine sim cao với "CEO OpenAI Sam Altman"
  - Direct text match: chunk "Sam Altman, OpenAI's CEO, addressed..." có embedding gần với query
  - LLM có context trực tiếp → trả lời chính xác

Flat RAG không phụ thuộc vào extraction quality — retrieves raw text.
```

### Bài Học & Đề Xuất Khắc Phục

1. **Short-term fix:** Tăng vector fallback từ k=4 lên k=8 cho GraphRAG khi graph context trống
2. **Medium-term:** Implement **Self-correction pipeline** (Bonus B): nếu graph context insufficient → tự động mở rộng hop, rồi fallback vector với k=8
3. **Long-term:** Cải thiện extraction quality — sử dụng model lớn hơn (llama-3.3-70b thay llama-3.1-8b) hoặc thêm few-shot examples vào EXTRACT_SYSTEM prompt
4. **Routing strategy:** Detect factoid questions (chứa "who is", "what is", "when was") → route sang Flat RAG; multi-hop/cross-doc → route sang GraphRAG

---

## Tổng Kết Pattern

| Pattern | Flat RAG | GraphRAG | Giải pháp |
|---------|----------|----------|-----------|
| Factoid đơn giản | Tốt | Dễ thất bại nếu entity miss | Routing: factoid → Flat RAG |
| Multi-hop | Thất bại | Tốt khi graph đủ | GraphRAG + Self-correction |
| Cross-doc | Tệ (fragments không link) | Tốt | GraphRAG preferred |
| Entity nổi tiếng | Tốt | Tốt | Both work |
| Entity hiếm/mới | Tệ (embedding drift) | Rất tệ nếu extraction miss | Cần manual verification |
| Temporal reasoning | Tệ | Tốt (published_date edge) | GraphRAG + date filter |
