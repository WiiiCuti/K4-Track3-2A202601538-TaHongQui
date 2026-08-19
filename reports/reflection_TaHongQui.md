# Suy Ngẫm Cá Nhân & Kế Hoạch Đồ Án — Lab 19: GraphRAG
## Reflection & Action Plan

**Học viên:** Tạ Hồng Quí — MSSV: 2A202601538  
**Khóa học:** AICB-K34 · Track 3: GraphRAG  
**Ngày thực hiện:** 19/08/2026

---

## Phần 1: Mapping Bài Giảng vào Code

| Khái niệm trong bài giảng | Module tương ứng | Hàm / Khối code cụ thể | Quan sát thực tế & Đánh giá |
|--------------------------|-----------------|------------------------|------------------------------|
| **Conservative Coreference** | Module 1 (M1) | `resolve_coref_single()`, `COREF_SYSTEM` prompt | Prompt "ONLY resolve when antecedent explicit in SAME chunk" giảm false positive; trade-off: nhiều `unresolved_mentions` hơn nhưng ít lỗi hơn |
| **Schema & Allowlist Guard** | Module 2 (M2) | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS`, `extract_triples()` | Allowlist loại bỏ 30-40% triples từ LLM; cần thiết để tránh garbage-in-garbage-out trong graph |
| **Bulk Cypher UNWIND** | Module 2 (M2) | `bulk_insert_nodes()`, `bulk_insert_edges()` | Batch 500 rows/call: nhanh hơn 50-100x so với individual MERGE; UNWIND là pattern chuẩn của Neo4j production |
| **Entity Resolution & Union-Find** | Module 3 (M3) | `build_resolution_map()`, `UnionFind`, `lexical_guard()` | Union-Find O(alpha(N)) gần như O(1) per operation; Lexical Guard quan trọng để chặn false merges như Apple vs Apple Music |
| **Super-node Degree Cap** | Module 4 (M4) | `retrieve_graph_context()`, `SUPER_NODE_DEGREE=100`, `SUPER_NODE_EDGE_CAP=50` | Google/Microsoft/OpenAI là super-nodes; cap giúp tránh context explosion; ORDER BY published_date DESC đảm bảo thông tin mới nhất |
| **LLM-as-a-Judge Evaluation** | Module 5 (M5) | `judge_answer()`, `JUDGE_SYSTEM` | 3 thang điểm (Comprehensiveness, Faithfulness, Multi-hop) cover đủ các khía cạnh; Rationale giải thích giúp debug |
| **BFS Traversal** | Module 4 (M4) | `retrieve_graph_context()` với `deque` | BFS với `frontier = deque()` đảm bảo explore by hop level; `GLOBAL_EDGE_CAP=250` tránh timeout |
| **Hybrid Context** | Module 4 (M4) | `answer_graph_rag()` | `=== GRAPH ===` + `=== VECTOR ===` cung cấp structured evidence + semantic coverage |
| **Provenance Citation** | Module 2+4 | `r.source_chunk_id`, `r.published_date` trong `textualize()` | Mỗi edge có trace về chunk gốc → explainable AI output |
| **Golden Dataset Evaluation** | Module 5 (M5) | `validate_golden()`, 3 groups: factoid/multi-hop/cross-doc | Phân nhóm câu hỏi giúp phân tích strengths/weaknesses của từng hệ thống |

---

## Phần 2: Quá Trình Debugging & Bài Học

### Lỗi Kỹ Thuật Phức Tạp Nhất

**Lỗi:** `json.JSONDecodeError` trong `parse_json_object()` khi LLM trả về JSON lẫn với markdown code fences.

**Biểu hiện:**
```
groq_json returned: "```json\n{\"triples\": [...]}\n```"
json.loads() throws JSONDecodeError on backticks
```

**Cách xử lý:**
1. Thêm fallback parsing với regex: `re.search(r"```(?:json)?\s*(\{.*?\})\s*```", text, re.DOTALL)`
2. Thêm fallback thứ 2: `re.search(r"\{.*\}", text, re.DOTALL)` để extract JSON object từ bất kỳ context nào
3. Return `{}` khi tất cả fail — không crash extraction pipeline

**Bài học:** LLM output không bao giờ 100% predictable — luôn cần defensive parsing với multiple fallbacks.

---

### Bài Học Từ Entity Resolution

**Vấn đề:** FAISS ANN trả về self-matches (entity A match với chính nó, sim=1.0)

**Fix:** Thêm điều kiện `if j <= i: continue` — skip các cặp đã xét và skip self-match (j==i)

**Bài học:** Khi build ANN index và search, luôn filter out self-matches và avoid double-counting pairs.

---

### Bài Học Từ Neo4j UNWIND

**Vấn đề:** Dynamic relationship types (FOUNDED_BY, INVESTED_IN...) không thể dùng với `$param` trong Cypher

**Sai (không hoạt động):**
```cypher
MERGE (s)-[r:$relation {}]->(t)  -- SYNTAX ERROR
```

**Đúng:**
```python
# Group by relation type, then build query per type
for rel, rows in by_rel.items():
    run_cypher(f"UNWIND $rows AS row ... MERGE (s)-[r:{rel} {{...}}]->(t)", rows=rows)
```

**Bài học:** Cypher không hỗ trợ dynamic relationship types qua parameters — phải dùng f-string interpolation với type grouping.

---

## Phần 3: Kế Hoạch Áp Dụng vào Đồ Án Thực Tế

### Tên Dự Án / Bài Toán
*[Điền tên đồ án thực tế của anh/chị]*

### Đánh Giá: Có Cần GraphRAG Không?

**Checklist để quyết định:**

| Đặc điểm bài toán | Cần GraphRAG? | Lý do |
|-------------------|--------------|-------|
| Câu hỏi chỉ cần 1 fact | Không | Flat RAG đủ |
| Câu hỏi cần chain 2+ entities/relations | Có | BFS traversal cần thiết |
| So sánh thông tin từ nhiều nguồn theo thời gian | Có | published_date + cross-doc |
| Dữ liệu có entity nhắc đến nhiều lần across docs | Có | Entity resolution giúp consolidate |
| Low latency requirement (<1s) | Không | GraphRAG ~3-5s |
| Cần explainability/traceability | Có | source_chunk_id provenance |

### Cấu Trúc Node & Relation Dự Kiến
*(Ví dụ cho bài toán phân tích tin tức kinh doanh)*

**Nodes:**
- `Company` (id, name, name_norm, sector, country)
- `Person` (id, name, name_norm, role)
- `Product` (id, name, category)
- `Event` (id, name, event_type, date)

**Relations:**
- `FOUNDED_BY` (Company -> Person)
- `CEO_OF` (Person -> Company)
- `LAUNCHED` (Company -> Product)
- `ACQUIRED` (Company -> Company)
- `INVESTED_IN` (Company -> Company)
- `PARTICIPATED_IN` (Company|Person -> Event)

### Chiến Lược Xử Lý Super-node & Entity Resolution

**Super-node strategy:**
- Detect nodes với degree > 50 (ngưỡng thấp hơn vì dataset nhỏ hơn)
- Cap: 30 edges mới nhất thay vì 50
- Add `relevance_score` vào edges để sort by relevance thay vì chỉ date

**Entity Resolution strategy:**
- Tăng threshold lên 0.92 nếu domain-specific entities (tránh merge công ty cùng ngành)
- Thêm manual aliases cho các công ty VN có tên viết tắt phổ biến (VCB, BID, CTG...)
- Near-dedup với MinHash trước khi chunk để giảm duplicate evidence

---

## Tự Đánh Giá

| Tiêu chí | Điểm tự chấm (1–5) | Ghi chú |
|----------|---------------------|---------|
| Mức độ hiểu bài giảng GraphRAG | 4 | Nắm được core concepts; cần thực hành thêm với dataset lớn hơn |
| Khả năng kiểm soát AI Coding Agent | 4 | Biết từ chối đề xuất O(N²); biết yêu cầu UNWIND pattern |
| Chất lượng đồ thị tri thức xây dựng | 3 | Phụ thuộc nhiều vào extraction quality của LLM |
| Khả năng phân tích và debug hệ thống | 4 | Đã debug được JSON parsing, Neo4j UNWIND, FAISS self-match |
