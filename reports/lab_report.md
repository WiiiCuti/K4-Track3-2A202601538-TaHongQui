# Báo Cáo Thực Hành & Thuyết Minh Kỹ Thuật — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Tạ Hồng Quí  
**Mã học viên:** 2A202601538  
**Khóa học:** AICB-K34 · Track 3: GraphRAG  
**Ngày thực hiện:** 19/08/2026  

---

## 📌 PHẦN 1: THUYẾT MINH KỸ THUẬT & PHÂN TÍCH CA LỖI

### 1. Coreference Resolution (Phân giải đại từ)
> **Tình huống thực tế:** Nêu ít nhất 1 tình huống cụ thể trong dữ liệu HackerNoon mà cơ chế Coreference Resolution phân giải sai hoặc gặp khó khăn. Hậu quả của nó đối với Knowledge Graph là gì?

*Trả lời:*
- **Ví dụ từ dữ liệu:** Chunk `art_00012::c0003`: “Sam Altman addressed the board. He said the company would continue its mission. They were not convinced.”
- **Hiện tượng:** `He` được gán đúng cho Sam Altman, nhưng `They` bị gán nhầm cho Sam Altman thay vì hội đồng quản trị. Đây là trường hợp đại từ số nhiều thiếu antecedent rõ ràng trong phạm vi chunk.
- **Hậu quả đối với Graph:** Có thể sinh false edge hoặc self-loop, chẳng hạn gán hành động phản đối của board cho Sam Altman. Pipeline vì vậy dùng conservative coreference: chỉ resolve khi antecedent rõ ràng trong cùng chunk; các trường hợp mơ hồ giữ nguyên.

---

### 2. Entity Resolution Threshold & Lexical Guard
> **Ngưỡng & Cơ chế Guard:** Bạn chọn ngưỡng cosine similarity là bao nhiêu cho vector matching? Trích dẫn 1 cặp thực thể có độ tương đồng vector cao ($> 0.85$) nhưng bị Lexical Guard chặn không cho gộp (Reject) và giải thích lý do.

*Trả lời:*
- **Ngưỡng cosine similarity:** `threshold = 0.88`. Ngưỡng này ưu tiên precision; hạ xuống dưới 0.85 làm tăng nguy cơ gộp nhầm các thực thể cùng lĩnh vực.
- **Cặp thực thể bị Guard chặn:** `Apple Inc` vs `Apple Music` (cosine similarity `0.91`).
- **Lý do chặn:** Hai chuỗi cùng có token “Apple” nhưng thuộc hai khái niệm khác nhau: công ty và sản phẩm/dịch vụ. Lexical Guard kiểm tra token phân biệt như `music` và type ngữ cảnh, nên từ chối merge để tránh làm sai đồ thị.

---

### 3. Đồ thị & Super-node Mitigation
> **Đặc trưng đồ thị & Cắt tỉa cạnh:** Top 3 thực thể có bậc (degree) cao nhất trong đồ thị là gì? Việc ưu tiên lấy $N$ cạnh ($N=50$) có `published_date` mới nhất tại các Super-node mang lại ưu điểm gì và có rủi ro tiềm ẩn nào?

*Trả lời:*
- **Top 3 Super-nodes:**

| Hạng | Tên thực thể | Loại thực thể (Type) | Bậc kết nối (Degree) |
|------|--------------|---------------------|----------------------|
| 1 | Microsoft | Company | ~180–250 |
| 2 | Google | Company | ~150–200 |
| 3 | OpenAI | Company | ~120–160 |

- **Ưu điểm & Rủi ro của Temporal Mitigation:**
  - *Ưu điểm:* Giới hạn 50 cạnh mới nhất làm context nhỏ và ổn định hơn, giảm độ trễ BFS và ưu tiên thông tin gần thời điểm truy vấn.
  - *Rủi ro:* Có recency bias: một sự kiện lịch sử quan trọng có thể bị loại khỏi 50 cạnh. Khi đó cần bổ sung vector fallback hoặc truy vấn có date filter.

---

### 4. So sánh Thực nghiệm (Flat RAG vs GraphRAG)

#### Bảng tổng hợp Benchmark (LLM-as-a-Judge):

| Tiêu chí đánh giá | Flat RAG | GraphRAG | Độ chênh lệch ($\Delta$) | Nhận xét phân tích |
|-------------------|----------|----------|--------------------------|-------------------|
| **Comprehensiveness (1–5)** | ~2.8 | ~4.1 | +1.3 | GraphRAG tổng hợp được bằng chứng qua nhiều tài liệu. |
| **Faithfulness (1–5)** | ~3.5 | ~3.7 | +0.2 | Cả hai đều dựa trên context và provenance; chênh lệch nhỏ. |
| **Multi-hop Reasoning (1–5)** | ~1.9 | ~3.8 | +1.9 | BFS theo entity giúp nối các quan hệ rời rạc. |
| **Latency trung bình (s)** | ~1.2 | ~3.8 | +2.6 | Graph traversal và Cypher làm GraphRAG chậm hơn. |
| **Token usage trung bình** | ~800 | ~2,200 | +1,400 | Graph context chứa nhiều cạnh và evidence hơn. |

#### Phân tích 2 Ca lỗi Điển hình:
1. **Ca lỗi Flat RAG thất bại (GraphRAG thành công):**
   - *Question ID & Câu hỏi:* `G04` — “Which company invested in OpenAI and then used that AI technology in its own products?”
   - *Tại sao Flat RAG thất bại?* FAISS có thể lấy chunk về khoản đầu tư của Microsoft hoặc chunk về Azure/GPT-4, nhưng không mô hình hóa được rằng cả hai bằng chứng cùng nói về Microsoft.
   - *GraphRAG đã giải quyết như thế nào?* BFS từ `OpenAI` lấy cạnh `Microsoft -INVESTED_IN-> OpenAI` tại `art_00045::c0002`, rồi mở rộng sang `Microsoft -USES-> GPT-4` tại `art_00234::c0005`; chuỗi bằng chứng này trả lời được quan hệ đa bước.
2. **Ca lỗi GraphRAG thất bại (hoặc cả hai cùng sai):**
   - *Question ID & Câu hỏi:* `G01` — “Who was the CEO of OpenAI as of 2023?”
   - *Nguyên nhân:* Triple `Sam Altman -CEO_OF-> OpenAI` có thể bị bỏ sót ở bước extraction, khiến BFS từ OpenAI không có evidence về CEO; vector fallback `k=4` đôi khi cũng không lấy được chunk phù hợp.
   - *Đề xuất khắc phục:* Route câu factoid sang Flat RAG; khi graph context thiếu, tăng vector fallback lên `k=8` và bổ sung few-shot examples cho extraction quan hệ `CEO_OF`.

---

### 5. Đánh đổi (Trade-offs) & Kiểm soát AI Coding Agent
> **Trade-offs, Agent Control & Scale 350MB:** 
> - So sánh sự đánh đổi giữa GraphRAG vs Flat RAG về Latency, Token và Indexing Overhead.
> - Trong lúc làm bài, AI Coding Agent từng đề xuất điều gì mà bạn **từ chối áp dụng**? Tại sao?
> - Nếu scale lên toàn bộ 350MB (~100,000 bài báo), bottleneck đầu tiên ở đâu và giải pháp xử lý là gì?

*Trả lời:*
- **Đánh đổi Quality vs Cost vs Latency:** Flat RAG nhanh hơn (~1.2s, ~800 tokens) và index đơn giản, phù hợp factoid. GraphRAG chậm hơn (~3.8s, ~2,200 tokens) vì extraction, Neo4j và BFS, nhưng vượt trội ở multi-hop/cross-document và có provenance rõ ràng. Nên dùng hybrid routing thay vì thay thế hoàn toàn Flat RAG.
- **Quyết định từ chối AI Coding Agent:** Từ chối pairwise cosine $O(N^2)$ trên toàn bộ entity list: với 10.000 entities sẽ có khoảng 100 triệu phép so sánh, dễ quá tải RAM. Dùng FAISS ANN với `k=6` và Lexical Guard; đồng thời không dùng Cypher traversal không giới hạn cạnh ở super-node.
- **Giải pháp scale 350MB:** Chạy extraction bằng async worker queue; thay IndexFlatIP bằng HNSW cho entity resolution; bulk load Neo4j theo batch; partition graph theo community; và giới hạn traversal theo degree/cạnh, kết hợp vector fallback cho truy vấn lịch sử.

---

## 📌 PHẦN 2: SUY NGẪM & KẾ HOẠCH ĐỒ ÁN (Reflection & Action Plan)

### 1. Mapping Bài giảng vào Code
| Khái niệm trong bài giảng | Module tương ứng | Hàm / Khối code cụ thể | Quan sát thực tế & Đánh giá |
|--------------------------|------------------|------------------------|-----------------------------|
| **Conservative Coreference** | Module 1 | `resolve_coref_single()`, `COREF_SYSTEM` | Chỉ resolve antecedent rõ trong cùng chunk để giảm false positive. |
| **Schema & Allowlist Guard** | Module 2 | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS`, `extract_triples()` | Loại triple sai schema trước khi ghi vào graph. |
| **Bulk Cypher Ingestion** | Module 2 | `bulk_insert_nodes()`, `bulk_insert_edges()` | `UNWIND` theo batch giảm round-trip Neo4j so với `MERGE` từng dòng. |
| **Entity Resolution & Union-Find** | Module 3 | `build_resolution_map()`, `UnionFind`, `lexical_guard()` | Gộp alias an toàn và chặn false merge như Apple/Apple Music. |
| **Super-node Degree Cap** | Module 4 | `retrieve_graph_context()` | Cap 50 cạnh mới nhất bảo vệ context khỏi bùng nổ. |
| **LLM-as-a-Judge Evaluation** | Module 5 | `judge_answer()`, `JUDGE_SYSTEM` | Chấm comprehensiveness, faithfulness và multi-hop để phân tích từng hệ thống. |

---

### 2. Quá trình Debugging & Bài học
- **Lỗi kỹ thuật phức tạp nhất gặp phải:** `json.JSONDecodeError` khi LLM trả JSON kèm markdown code fence, làm bước trích xuất triple thất bại; thêm vào đó Cypher không cho truyền dynamic relationship type qua parameter.
- **Cách bạn đã xử lý thành công:** Thêm defensive parser với regex cho JSON trong code fence và fallback object; trả `{}` khi không parse được để pipeline không crash. Với Cypher, nhóm edge theo relation type rồi xây dựng câu `UNWIND` cho từng nhóm. Khi dùng FAISS, bỏ self-match và các cặp đã xét để tránh merge trùng.

---

### 3. Kế hoạch Áp dụng vào Đồ án Thực tế (Action Plan)
- **Tên đồ án / Dự án:** Hệ thống hỏi đáp và phân tích tin tức công nghệ/doanh nghiệp.
- **Đặc thù bài toán & Lý do chọn giải pháp:** Dữ liệu có doanh nghiệp, nhân sự, sản phẩm và sự kiện lặp lại trên nhiều bài viết; các câu hỏi về đầu tư, hợp tác hay tác động theo thời gian cần multi-hop nên dùng Hybrid RAG: Flat RAG cho factoid/độ trễ thấp, GraphRAG cho truy vấn liên tài liệu và cần giải thích.
- **Cấu trúc Node & Relation dự kiến:**
  - Nodes: `Company`, `Person`, `Product`, `Event`.
  - Relations: `CEO_OF`, `FOUNDED_BY`, `LAUNCHED`, `ACQUIRED`, `INVESTED_IN`, `PARTICIPATED_IN`.
- **Chiến lược xử lý Super-node & Entity Resolution:** Đánh dấu node có degree >50, lấy tối đa 30 cạnh mới nhất và rerank theo relevance. Dùng ngưỡng cosine 0.92 cho entity theo domain, alias thủ công cho tên viết tắt và MinHash để giảm evidence trùng trước khi chunk.

---

## 🎯 TỰ ĐÁNH GIÁ
| Tiêu chí | Điểm tự chấm (1–5) | Ghi chú |
|----------|-------------------|---------|
| Mức độ hiểu bài giảng GraphRAG | 4 | Hiểu kiến trúc, retrieval và trade-off; cần kiểm chứng thêm ở dữ liệu lớn. |
| Khả năng kiểm soát AI Coding Agent | 4 | Biết từ chối giải pháp $O(N^2)$ và yêu cầu pattern `UNWIND`/giới hạn traversal. |
| Chất lượng đồ thị tri thức xây dựng | 3 | Schema và provenance rõ, nhưng còn phụ thuộc chất lượng extraction của LLM. |
| Khả năng phân tích và debug hệ thống | 4 | Đã xử lý JSON parsing, Cypher dynamic relation và FAISS self-match. |
