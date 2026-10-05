# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Nguyễn Khắc Quang  **MSSV:** 2A202602885  **Ngày:** 05/10/2026

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu được trích xuất chính xác 100% từ lần chạy thực nghiệm hoàn chỉnh ghi trong [ket_qua_benchmark_kg.txt](file:///c:/Users/PC/K4-Track3-Day19-NguyenKhacQuang-2A202602885-GraphRAG-Knowledge-Graphs/ket_qua_benchmark_kg.txt). Bản thiết kế ontology nộp riêng ở [report/ONTOLOGY.md](file:///c:/Users/PC/K4-Track3-Day19-NguyenKhacQuang-2A202602885-GraphRAG-Knowledge-Graphs/report/ONTOLOGY.md).

---

## 1. Chi phí (10 điểm)

Hai bảng `Indexing` và `Querying` trích xuất nguyên văn từ [ket_qua_benchmark_kg.txt](file:///c:/Users/PC/K4-Track3-Day19-NguyenKhacQuang-2A202602885-GraphRAG-Knowledge-Graphs/ket_qua_benchmark_kg.txt):

```text
Chat model: gemini:gemini-3.5-flash-lite | Embedding: gemini:gemini-embedding-001 | top_k=3 | chunk_size=800 | chunks=176 | KG: 203 nodes / 382 rels

== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176         0        0   0.00000    111.3
graph       196     34619     5743   0.00576    166.7

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.51   1.33      696       71   0.00010     1.42
graph       0.94   1.83     5732      175   0.00064    10.19
```

### Bảng đối chiếu chỉ số và tỉ lệ Graph / Flat:

| Chỉ số | Flat RAG | GraphRAG | Tỉ lệ (Graph / Flat) | Phân tích chi tiết |
| :--- | :--- | :--- | :--- | :--- |
| **Indexing USD** | $0.00000 | $0.00576 | **+0.00576 USD** | Flat chỉ tính vector embedding (miễn phí theo bậc Free); Graph tốn thêm phí LLM trích xuất tin tức. |
| **Indexing giây** | 111.3s | 166.7s | **× 1.50** | Tăng thêm 55.4 giây do gọi tuần tự 20 lần LLM trích xuất JSON các bài báo tin tức. |
| **Mỗi câu: USD** | $0.00010 | $0.00064 | **× 6.40** | GraphRAG tốn gấp 6.4 lần tiền mỗi câu hỏi do ngữ cảnh mở rộng dài hơn. |
| **Mỗi câu: giây** | 1.42s | 10.19s | **× 7.18** | GraphRAG cần duyệt multi-hop trên Neo4j và LLM mất nhiều thời gian hơn để đọc 5.7k tokens. |
| **Mỗi câu: in_tok**| 696 tokens | 5,732 tokens| **× 8.24** | GraphRAG bổ sung các facts từ đồ thị (Điều luật, Khoản phạt, Vụ án liên quan). |

### Chi phí tăng thêm đến từ đâu?
1. **Ở pha Indexing (Dựng hệ thống):** Toàn bộ chi phí USD tăng thêm ($0.00576) xuất phát từ **20 lượt gọi LLM** (`34,619 input tokens` và `5,743 output tokens`) để đọc 20 bài báo văn xuôi tự do và chuyển đổi thành cấu trúc JSON (các node `Case`, `Person`, `Substance`, `Location` và các liên kết `CHARGED_WITH`, `INVOLVES`). Trong khi đó, văn bản luật được phân tích bằng **Regex thuần túy nên tốn 0 USD**.
2. **Ở pha Querying (Truy vấn mỗi câu):** Chi phí tăng gấp 6.4 lần là do **chiều dài của prompt ngữ cảnh**. Flat RAG chỉ ghép `top_k=3` chunk vector (~696 tokens). GraphRAG thực hiện duyệt đồ thị qua node cầu nối `Crime` để lấy thêm toàn bộ các Điều luật và Khoản luật điều chỉnh, đẩy số lượng input tokens trung bình lên `5,732 tokens`.

### Ước tính điểm hòa vốn (Break-even Analysis):
- **Góc nhìn 1: Chi phí token thuần túy (Cost Parity):**
  GraphRAG có chi phí cố định ban đầu cao hơn ($0.00576 > $0) và chi phí biến đổi mỗi câu hỏi cũng cao hơn ($0.00064 > $0.00010). Do đó, nếu chỉ xét riêng tiền trả cho API tokens, GraphRAG **không bao giờ rẻ hơn Flat RAG**.
- **Góc nhìn 2: Điểm hòa vốn theo Chi phí sai sót & Giá trị thông tin (ROI & Risk Break-even):**
  Flat RAG trả lời sai hoặc thiếu thông tin ở 50% số câu hỏi (Recall chỉ đạt 0.51, đặc biệt các câu hỏi nghiệp vụ xuyên nguồn chỉ đạt 0.33 và câu tổng hợp đạt 0.00). Trong lĩnh vực pháp lý, trả lời sai một khung hình phạt (ví dụ: bị cáo đối diện tử hình nhưng hệ thống báo chỉ bị phạt 2–7 năm tù) sẽ gây thiệt hại khôn lường về trách nhiệm pháp lý hoặc mất hàng giờ kiểm tra lại của luật sư. Với chi phí đầu tư ban đầu vỏn vẹn **$0.00576 USD** (chưa tới 150 VNĐ), GraphRAG đưa Recall lên **0.94** và Judge score lên **1.83/2.0**. Điểm hòa vốn nghiệp vụ đạt được **ngay ở câu hỏi đầu tiên** ngăn chặn được một phán đoán sai sót!
- **Góc nhìn 3: So sánh với Long-Context Brute-force RAG:**
  Nếu không dùng đồ thị mà nhồi toàn bộ 38 tài liệu (khoảng 85,000 tokens) vào cửa sổ ngữ cảnh của LLM cho mỗi câu hỏi, chi phí mỗi câu hỏi sẽ lên tới ~$0.012 USD. GraphRAG ($0.00064 USD/câu) tiết kiệm được $0.01136 USD mỗi câu so với Long-context RAG. Điểm hòa vốn chi phí giữa GraphRAG và Long-context RAG xảy ra tại:
  $$\text{Break-even} = \frac{\$0.00576}{\$0.01200 - \$0.00064} \approx 0.51 \text{ câu hỏi}$$
  Nghĩa là **ngay từ câu hỏi thứ nhất**, GraphRAG đã tiết kiệm chi phí hơn phương án brute-force long-context!

---

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại câu hỏi | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu ngắn gọn) |
| :---: | :---: | :---: | :---: | :---: | :--- |
| **Q1** | `single-hop-law` | 1.00 / 2 | 1.00 / 2 | **Hòa** | Định nghĩa tiền chất nằm gọn trong một đoạn văn (Điều 2 Luật PCMT) nên vector search lấy đủ ngữ cảnh mà không cần đồ thị. |
| **Q2** | `single-hop-news` | 1.00 / 2 | 1.00 / 2 | **Hòa** | Tên 2 bị cáo lãnh án tử hình nằm trọn vẹn trong một bài báo tin tức vụ án 36kg ma túy, Flat RAG tìm trúng chunk dữ liệu. |
| **Q3** | `cross-kb` | 0.33 / 1 | 1.00 / 2 | **Graph** | Flat RAG chỉ tìm thấy mức án 36 tháng mà thiếu điều luật; GraphRAG đi qua node tội danh `Crime` sang Điều 251 khoản 1 để trả lời đầy đủ. |
| **Q4** | `cross-kb` | 0.33 / 1 | 0.67 / 1 | **Graph** | Flat RAG hoàn toàn không biết điều luật áp dụng; GraphRAG tìm ra Điều 255 BLHS và hành vi tổ chức sử dụng nhờ liên kết đồ thị. |
| **Q5** | `cross-kb-multi-hop` | 0.40 / 1 | 1.00 / 2 | **Graph** | Flat RAG thiếu căn cứ pháp lý; GraphRAG kết nối tang vật 9.6kg MDMA trong vụ án sang điểm b khoản 4 Điều 250 để nêu đúng khung phạt 20 năm, chung thân hoặc tử hình. |
| **Q6** | `aggregation` | 0.00 / 1 | 1.00 / 2 | **Graph** | Flat RAG chỉ lấy được 3 chunk ngẫu nhiên và bỏ sót phần lớn các vụ án; GraphRAG truy vấn tổng hợp đầy đủ 4 vụ án liên quan đến MDMA. |

### Quy luật tổng quát liên hệ giữa "Loại câu hỏi" và "Bên thắng":
1. **Câu hỏi Single-hop (Cục bộ trong 1 KB):** `Flat RAG == GraphRAG`. Khi toàn bộ câu trả lời nằm trọn trong một đoạn văn bản cục bộ, Flat RAG tối ưu hơn vì đạt độ chính xác tương đương nhưng tốc độ nhanh hơn gấp 7 lần và chi phí rẻ hơn gấp 6 lần.
2. **Câu hỏi Cross-KB (Xuyên nguồn giữa Tin tức và Luật):** `GraphRAG >> Flat RAG`. Flat RAG luôn thất bại (Recall giảm xuống 0.33–0.40) vì không có đoạn văn nào trong thực tế đồng thời chứa cả thông tin bị cáo đời thực lẫn khung hình phạt quy định trong văn bản luật. GraphRAG áp đảo nhờ khả năng đi xuyên qua node cầu nối `Crime`.
3. **Câu hỏi Aggregation (Tổng hợp phân tán toàn cục):** `GraphRAG áp đảo tuyệt đối`. Vector search bị giới hạn bởi tham số `top_k` (chỉ lấy 3–5 chunk tương đồng ngữ nghĩa nhất), do đó không thể gom đủ các thực thể nằm rải rác trên 20 bài báo khác nhau. Ngược lại, đồ thị cho phép thực hiện phép gom nhóm tự nhiên: `(Case)-[:INVOLVES]->(Substance {name: 'MDMA'})`.

---

## 3. Phân tích lỗi (20 điểm)

### Lỗi E2: Thiếu ngữ cảnh luật khi hỏi khung hình phạt tối đa (Câu Q4)
- **Nằm ở bước nào của pipeline:** Bước trích xuất ngữ cảnh đồ thị [`Neo4jGraph.context()`](file:///c:/Users/PC/K4-Track3-Day19-NguyenKhacQuang-2A202602885-GraphRAG-Knowledge-Graphs/src/graph.py#L280) (KG-3).
- **Hiện tượng:** Ở câu Q4 (*"Giang hồ 'Hoàng Nato' bị bắt về hành vi gì, và hành vi đó có thể bị phạt tù tối đa bao nhiêu theo Bộ luật Hình sự?"*), GraphRAG xác định đúng Điều 255 BLHS nhưng lại trả lời mức phạt tối đa là **07 năm tù** (theo Khoản 1), trong khi mức phạt tù tối đa thực tế theo Bộ luật Hình sự là **20 năm hoặc tù chung thân** (quy định tại Khoản 4).
- **Bằng chứng:** 
  1. *Nguyên văn câu trả lời của GraphRAG (trích từ `ket_qua_benchmark_kg.txt`):*
     > `Theo Điều 255 BLHS - Tội tổ chức sử dụng trái phép chất ma túy (Khoản 1 quy định mức phạt tù từ 02 năm đến 07 năm đối với người phạm tội cơ bản), ngữ cảnh không nêu rõ các tình tiết tăng nặng định khung cụ thể nên mức phạt tù tối đa theo khung cơ bản được đề cập là 07 năm tù (Lưu ý: Ngữ cảnh không cung cấp đầy đủ các khoản tăng nặng khác của Điều 255).`
  2. *Truy vấn Cypher kiểm chứng cấu trúc các khoản của Điều 255 trong database:*
     ```cypher
     MATCH (a:Article {id: "Điều 255 BLHS"})-[:HAS_CLAUSE]->(cl:Clause)
     RETURN cl.number AS number, cl.penalty AS penalty
     ORDER BY number;
     ```
  3. *Kết quả thực tế từ Neo4j:*
     ```text
     ╒════════╤═════════════════════════════════════════════╕
     │"number"│"penalty"                                    │
     ╞════════╪═════════════════════════════════════════════╡
     │1       │"phạt tù từ 02 năm đến 07 năm"               │
     ├────────┼─────────────────────────────────────────────┤
     │2       │"phạt tù từ 07 năm đến 15 năm"               │
     ├────────┼─────────────────────────────────────────────┤
     │3       │"phạt tù từ 15 năm đến 20 năm"               │
     ├────────┼─────────────────────────────────────────────┤
     │4       │"phạt tù hai mươi năm hoặc tù chung thân"    │
     └────────┴─────────────────────────────────────────────┘
     ```
- **Nguyên nhân:** Trong hàm `context()` của `src/graph.py`, câu truy vấn Cypher được tối ưu hóa để hạn chế tràn ngữ cảnh bằng điều kiện:
  `WHERE cl.number = 1 OR EXISTS { (k)-[:INVOLVES]->(:Substance)<-[:MENTIONS]-(cl) }`.
  Chiến lược này hoạt động hoàn hảo cho các vụ án có tang vật rõ ràng (như Q5 có MDMA để kéo Khoản 4). Tuy nhiên, trong vụ án của Hoàng Nato, bài báo chỉ nêu hành vi tổ chức sử dụng mà không nêu rõ định lượng chất ma túy. Do không khớp được chất nào, Cypher chỉ trả về duy nhất Khoản 1 (khung phạt cơ bản), khiến LLM bị tước đoạt thông tin về Khoản 4 (khung tăng nặng cao nhất).
- **Đề xuất sửa:** 
  Bổ sung nhận diện câu hỏi có từ khóa cực trị (`"tối đa"`, `"cao nhất"`, `"khung hình phạt cao nhất"`) để kéo bổ sung khoản có số thứ tự cao nhất của Điều luật:
  ```cypher
  MATCH (a:Article)-[:HAS_CLAUSE]->(cl:Clause)
  WHERE a.id = $article_id
  WITH cl ORDER BY cl.number DESC LIMIT 1
  RETURN cl.id, cl.number, cl.text
  ```

---

### Lỗi E3: Trùng thực thể (Entity Duplication) do tên vụ án do LLM tự đặt

- **Nằm ở bước nào của pipeline:** Thiết kế Ontology (Mục 2) và Pha trích xuất LLM [`build_graph()`](file:///c:/Users/PC/K4-Track3-Day19-NguyenKhacQuang-2A202602885-GraphRAG-Knowledge-Graphs/src/graph.py#L329) (KG-2).
- **Hiện tượng:** Cùng một vụ án xét xử đối tượng Cái Quang Huy vận chuyển ma túy qua sân bay Nội Bài được 2 bài báo đưa tin, nhưng hệ thống tạo ra **hai node `Case` riêng biệt** trong đồ thị thay vì hợp nhất.
- **Bằng chứng:** 
  1. *Truy vấn Cypher tìm các vụ án liên quan đến sân bay Nội Bài hoặc Đức:*
     ```cypher
     MATCH (k:Case)
     WHERE toLower(k.name) CONTAINS "nội bài" OR toLower(k.name) CONTAINS "đức"
     RETURN k.name AS case_name, k.doc_id AS doc_id, k.date AS date;
     ```
  2. *Kết quả thực tế từ Neo4j:*
     ```text
     ╒══════════════════════════════════════════════════════════════════╤══════════════════════════╤════════════╕
     │"case_name"                                                       │"doc_id"                  │"date"      │
     ╞══════════════════════════════════════════════════════════════════╪══════════════════════════╪════════════╡
     │"Vụ vận chuyển ma túy từ Đức về Việt Nam qua sân bay Nội Bài"     │"news-100260905141147814" │"2023-09-05"│
     ├──────────────────────────────────────────────────────────────────┼──────────────────────────┼────────────┤
     │"Vụ vận chuyển ma túy qua sân bay Nội Bài do Cái Quang Huy thực hiện"│"news-100260905183424109"│"2023-09-05"│
     └──────────────────────────────────────────────────────────────────┴──────────────────────────┴────────────┘
     ```
- **Nguyên nhân:** Khóa định danh của nhãn `Case` trong Neo4j được thiết kế là `MERGE (k:Case {name: $name})`. Chuỗi `name` được trích xuất tự do từ LLM cho từng bài báo riêng biệt. Do phóng viên giật tít và hành văn khác nhau, LLM đã đặt ra hai tên vụ án khác nhau (`"Vụ vận chuyển ma túy từ Đức về Việt Nam qua sân bay Nội Bài"` và `"Vụ vận chuyển ma túy qua sân bay Nội Bài do Cái Quang Huy thực hiện"`). Do `name` không trùng khớp chính xác 100%, Neo4j coi đây là hai thực thể tách rời, dẫn tới việc tang vật (khối lượng MDMA, Ketamine) và các bị cáo liên quan bị phân tán thành hai cụm nhỏ.
- **Đề xuất sửa:** 
  1. **Cải tiến Ontology:** Không dùng `name` đơn thuần làm khóa định danh duy nhất cho `Case`. Cần dùng khóa phức hợp (composite key): `date + location + primary_substance`.
  2. **Bổ sung Entity Resolution Pipeline:** Trước khi nạp vào Neo4j, tính toán độ tương đồng cosine giữa embedding tóm tắt (`summary`) của các vụ án xảy ra cùng ngày tại cùng một địa bàn. Nếu similarity > 0.85, thực hiện hợp nhất danh sách đối tượng và tang vật vào cùng một node `Case`.

---

## 4. Kết luận (5 điểm)

Dựa trên toàn bộ số liệu thực nghiệm đo đạc:

1. **Khi nào NÊN dùng Knowledge Graph (GraphRAG):**
   - **Dữ liệu phân tán nhiều nguồn độc lập (Heterogeneous/Cross-domain KBs):** Khi hệ thống cần trả lời các câu hỏi kết nối giữa "sự kiện thực tế" (tin tức, hồ sơ khách hàng, nhật ký hệ thống) với "quy chuẩn trừu tượng" (văn bản quy phạm pháp luật, chính sách bảo hiểm, quy trình kỹ thuật). Minh chứng là ở câu Q3 và Q5, GraphRAG đưa Recall từ 0.33–0.40 lên **1.00**.
   - **Bài toán truy vết và tổng hợp toàn cục (Global Aggregation):** Khi người dùng hỏi dạng gom nhóm *"Có những vụ việc nào liên quan đến X?"* (câu Q6), GraphRAG đạt Recall **1.00** trong khi Flat RAG hoàn toàn thất bại (Recall = **0.00**).
   - **Yêu cầu tính giải trình cao (Explainability):** GraphRAG cung cấp đường dẫn tường minh: `Person → Case → Crime → Article → Clause`, giúp người dùng kiểm chứng chính xác căn cứ pháp lý thay vì tin vào xác suất embedding.

2. **Khi nào Flat RAG LÀ ĐỦ:**
   - **Câu hỏi đơn bước (Single-hop):** Khi câu hỏi tập trung vào định nghĩa, sự kiện nằm gọn trong một tài liệu duy nhất (như Q1 và Q2, cả hai pipeline đều đạt Recall 1.00 và Judge = 2).
   - **Hệ thống có ràng buộc khắt khe về độ trễ và ngân sách:** Flat RAG có độ trễ chỉ **1.42 giây** (nhanh hơn 7.18 lần so với 10.19s của GraphRAG) và chi phí chỉ **$0.00010/câu** (rẻ hơn 6.4 lần so với $0.00064). Nếu 90% câu hỏi của người dùng là tra cứu cục bộ, việc đầu tư Knowledge Graph sẽ gây lãng phí tài nguyên tính toán.

---

## 5. Tự kiểm (5 điểm)

### Output chạy unit test:
```text
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.05s
```

### Output chạy tự kiểm tra hợp đồng đồ thị:
```text
$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = gemini:gemini-3.5-flash-lite | embedding = gemini:gemini-embedding-001
[OK] KG-2 build_graph: 148 node / 294 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 23 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00056. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

### Danh mục 3 ảnh Neo4j Browser:
1. `report/img/kg_count.png`: Ảnh chụp bảng đếm đầy đủ 7 nhãn node trong Neo4j Browser (Article, Clause, Crime, Case, Person, Substance, Location).
2. `report/img/kg_cross_kb.png`: Ảnh chụp đồ thị đường đi xuyên 2 KB qua node cầu nối `Crime` (kết nối giữa `Person`/`Case` và `Article`).
3. `report/img/kg_my_case.png`: Ảnh chụp cụm đồ thị của đối tượng **Cái Quang Huy** (kết nối từ bị cáo Cái Quang Huy → Vụ án vận chuyển ma túy qua Nội Bài → Tội danh → Điều 250 BLHS, cùng tang vật MDMA, Ketamine và địa điểm Hà Nội).

**Người đã chọn cho `kg_my_case.png`:** Cái Quang Huy
