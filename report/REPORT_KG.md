# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Vũ Minh Hiếu  **MSSV:** 2A202602779  **Ngày:** 05/10/2026

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

---

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112    291.2
graph       196     91958     4712   0.00933    383.5

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       47   0.00013     6.01
graph       0.63   1.33     3318       83   0.00054     6.85
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | $0.00112 | $0.00933 | ×8.33 |
| Indexing giây | 291.2 | 383.5 | ×1.32 |
| Mỗi câu: USD | $0.00013 | $0.00054 | ×4.15 |
| Mỗi câu: giây | 6.01 | 6.85 | ×1.14 |
| Mỗi câu: in_tok | 694 | 3318 | ×4.78 |

**Chi phí tăng thêm đến từ đâu?**
1. **Lúc Indexing (One-off):** Chi phí GraphRAG cao gấp **8.33 lần** ($0.00933 so với $0.00112) do phải gọi thêm 20 lượt LLM chat để đọc và trích xuất cấu trúc quan hệ từ 20 bài báo tin tức thành JSON (tốn thêm 35,886 input tokens và 4,712 output tokens), trong khi Flat RAG chỉ tốn phí embedding văn bản thuần túy.
2. **Lúc Querying (Mỗi câu hỏi):** Chi phí GraphRAG cao gấp **4.15 lần** ($0.00054 so với $0.00013) vì mỗi lượt truy vấn, đồ thị tri thức cung cấp thêm các dữ kiện liên kết đa bước (`facts` gồm các Điều luật, khung hình phạt, tóm tắt vụ án). Điều này đẩy dung lượng prompt đầu vào từ 694 tokens lên 3,318 tokens/câu (gấp 4.78 lần), đổi lại hệ thống có được ngữ cảnh liên kết chính xác xuyên suốt giữa tin tức và điều luật tương ứng.

---

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | :---: | :---: | :---: | --- |
| **Q1** | single-hop-law | 1.00 / 2 | 1.00 / 2 | **Hòa** | Định nghĩa tiền chất nằm gọn trong Điều 2 Luật PCMT 2021 nên vector search của Flat RAG lấy trúng chunk và cả hai trả lời hoàn hảo. |
| **Q2** | single-hop-news | 1.00 / 2 | 1.00 / 2 | **Hòa (Graph chi tiết hơn)** | Cả hai đều tìm được 2 bị cáo nhận án tử hình từ bài báo về vụ 36kg ma túy; GraphRAG còn trích dẫn chính xác thêm Điều 251 BLHS. |
| **Q3** | cross-kb | 0.00 / 0 | 1.00 / 2 | **Graph** | Flat RAG không có chunk nào chứa cả mức án trong tin tức và khung hình phạt luật nên trả về "Không đủ thông tin", còn GraphRAG kết nối xuyên 2 KB qua node Crime trả lời chính xác trọn vẹn. |
| **Q4** | cross-kb | 0.00 / 0 | 0.00 / 0 | **Hòa (Chưa đạt)** | Flat RAG thiếu liên kết luật; GraphRAG tìm được vụ việc và Điều 255 nhưng do bộ lọc ngữ cảnh chỉ lấy khoản 1 (2-7 năm) trong khi câu hỏi hỏi mức "tối đa" (thuộc khoản 4), LLM nhận thấy thiếu khoản phạt cao nhất nên trả về "Không đủ thông tin". |
| **Q5** | cross-kb-multi-hop | 0.60 / 1 | 0.80 / 1 | **Graph** | GraphRAG lần theo đường Người → Vụ án (hơn 9.6kg MDMA) → Điều 250 → Khoản 4 và khung phạt 20 năm, chung thân hoặc tử hình, trong khi Flat RAG trích xuất nhầm sang "khoản b)". |
| **Q6** | aggregation | 0.00 / 1 | 0.00 / 1 | **Graph (Bao quát hơn)** | Cả hai đều chỉ ra đúng 3 vụ án có tang vật MDMA; GraphRAG nổi trội hơn khi tổng hợp bổ sung toàn bộ các Điều luật liên quan tới MDMA trong BLHS (Điều 249, 250, 251, 252). |

---

## 3. Phân tích lỗi (20 điểm)

### Lỗi E2: Thiếu ngữ cảnh luật (Missing Legal Context — Lọc khoản bỏ sót khung tối đa ở câu Q4)

- **Hiện tượng:** Câu hỏi Q4 hỏi về hành vi của *giang hồ 'Hoàng Nato'* và mức phạt tù **tối đa** theo Bộ luật Hình sự. GraphRAG đã xác định được đối tượng Dương Minh Tuấn ('Hoàng Nato') và tìm ra tội danh *tổ chức sử dụng trái phép chất ma túy* quy định tại Điều 255 BLHS, nhưng câu trả lời cuối cùng lại là *"Không đủ thông tin"*, dẫn đến recall = 0.00 và judge = 0.
- **Bằng chứng:**
  - Trích nguyên văn output câu Q4 từ `ket_qua_benchmark_kg.txt`:
    ```
    --- Q4 [cross-kb] graph recall=0.00 judge=0 3.41s
    Không đủ thông tin.
    ```
  - Kiểm tra hàm `Neo4jGraph.context` trên thực tế cho thấy các dữ kiện được đưa vào prompt chỉ bao gồm:
    ```
    [Điều 255 BLHS - Tội tổ chức sử dụng trái phép chất ma túy] khoản 1: 1. Người nào tổ chức sử dụng trái phép chất ma túy dưới bất kỳ hình thức nào, thì bị phạt tù từ 02 năm đến 07 năm.
    ```
  - Truy vấn Cypher kiểm tra cấu trúc toàn bộ các khoản của Điều 255 BLHS:
    ```cypher
    MATCH (a:Article {id: 'Điều 255 BLHS'})-[:HAS_CLAUSE]->(cl:Clause)
    RETURN cl.number AS khoán, cl.penalty AS muc_phat ORDER BY khoán;
    ```
    *Kết quả trong Neo4j:*
    ```
    khoan: 1 | muc_phat: "phạt tù từ 02 năm đến 07 năm"
    khoan: 2 | muc_phat: "phạt tù từ 07 năm đến 15 năm"
    khoan: 3 | muc_phat: "phạt tù từ 15 năm đến 20 năm"
    khoan: 4 | muc_phat: "phạt tù 20 năm hoặc tù chung thân"
    ```
- **Nguyên nhân:**
  Trong thiết kế truy vấn multi-hop của `Neo4jGraph.context`:
  ```cypher
  WHERE cl.number = 1 OR matched_substances > 0
  ```
  Quy tắc lọc này chỉ giữ lại **khoản 1** (khung cơ bản) và các khoản có `MENTIONS` chất ma túy mà vụ án đó `INVOLVES`. Trong bài báo về Hoàng Nato, chất được sử dụng là *etomidate* (không nằm trong danh sách các chất định lượng cụ thể của Điều 255, vốn định khung theo số lượng người tổ chức và địa điểm vi phạm). Do đó, khoản 4 (quy định mức án cao nhất: tù 20 năm hoặc tù chung thân) bị loại bỏ khỏi context. Prompt yêu cầu: *"Trả lời câu hỏi chỉ dựa trên ngữ cảnh... Nếu ngữ cảnh không đủ, nói không đủ thông tin"*, nên khi không có khoản 4 chứa mức phạt tối đa, LLM đã từ chối trả lời một cách có căn cứ.
- **Đề xuất sửa:**
  Khi câu hỏi chứa các từ khóa về mức phạt tối đa/khung cao nhất (như `"tối đa"`, `"cao nhất"`, `"nặng nhất"`), mở rộng Cypher query để lấy thêm khoản có số hiệu lớn nhất (`cl.number` lớn nhất) hoặc khoản có mức phạt chứa cụm `"tù chung thân"` / `"tử hình"` của Điều luật đó:
  ```cypher
  WITH a, max(cl.number) AS max_clause
  MATCH (a)-[:HAS_CLAUSE]->(cl:Clause)
  WHERE cl.number = 1 OR cl.number = max_clause OR matched_substances > 0
  RETURN DISTINCT a.id, cl.number, cl.text
  ```
  *Đánh đổi:* Thêm 1 khoản vào context tốn thêm khoảng 50–100 tokens cho mỗi truy vấn liên quan đến Điều luật đó.

---

### Lỗi E3: Trùng thực thể (Duplicate Entities — Phân mảnh node do chữ hoa/thường và từ đồng nghĩa)

- **Hiện tượng:** Trên đồ thị Neo4j, cùng một loại chất ma túy nhưng bị phân tách thành nhiều node `Substance` khác nhau, khiến các quan hệ `INVOLVES` từ tin tức và `MENTIONS` từ luật không quy tụ về cùng một điểm chung, làm suy giảm khả năng liên kết dữ liệu.
- **Bằng chứng:**
  Truy vấn Cypher kiểm tra danh sách các chất ma túy trên đồ thị:
  ```cypher
  MATCH (s:Substance) RETURN s.name AS name ORDER BY toLower(s.name);
  ```
  *Kết quả thực tế trong Neo4j:*
  ```
  {'name': 'Amphetamine'}
  {'name': 'chất ma túy'}
  {'name': 'Cocaine'}
  {'name': 'côca'}
  {'name': 'cần sa'}
  {'name': 'etomidate'}
  {'name': 'Heroine'}
  {'name': 'Ketamine'}
  {'name': 'ketamine'}
  {'name': 'ma túy'}
  {'name': 'ma túy tổng hợp'}
  {'name': 'MDMA'}
  {'name': 'Methamphetamine'}
  {'name': 'methamphetamine'}
  {'name': 'thuốc lắc'}
  {'name': 'thuốc phiện'}
  {'name': 'XLR-11'}
  ```
  Quan sát thấy:
  1. Phân mảnh hoa/thường: `Ketamine` và `ketamine`, `Methamphetamine` và `methamphetamine`.
  2. Phân mảnh tên lóng/tên thương mại và danh pháp: `MDMA` và `thuốc lắc`.
  3. Node rác ngữ nghĩa chung chung do LLM trích xuất tự do từ tin tức: `chất ma túy`, `ma túy`, `ma túy tổng hợp`.
- **Nguyên nhân:**
  Trong lệnh `add_news_case`, Cypher sử dụng lệnh:
  ```cypher
  MERGE (sub:Substance {name: s.name})
  ```
  Neo4j phân biệt chữ hoa và chữ thường trong thuộc tính `name` (case-sensitive). Khi văn bản luật tạo node với tên viết hoa chuẩn (`Ketamine`), còn LLM trích xuất từ bài báo viết chữ thường (`ketamine`), Neo4j tạo thành 2 node tách biệt. Đồng thời, danh sách chất chưa được cho qua một hàm chuẩn hóa `link_entity(s.name, SUBSTANCES)` trước khi ghi vào cơ sở dữ liệu.
- **Đề xuất sửa:**
  1. Áp dụng chuẩn hóa `link_entity` đối với danh mục chất ma túy `SUBSTANCES` ngay tại bước `extract_news_cases` trong `src/graph.py` tương tự như đã làm với tội danh (`crimes`).
  2. Sử dụng thuộc tính chuẩn hóa viết hoa chữ cái đầu hoặc chữ thường thống nhất trước khi `MERGE`:
     ```python
     substance_name = link_entity(s.get("name"), SUBSTANCES) or s.get("name").strip().capitalize()
     ```
  3. Lọc bỏ các từ mang nghĩa chung chung như *"ma túy"*, *"chất ma túy"* khỏi danh sách trích xuất của bài báo.
  *Đánh đổi:* Tốn thêm vài phép so khớp chuỗi trong Python lúc indexing (chi phí tính toán không đáng kể, thời gian dưới 1 giây cho 20 bài báo).

---

## 4. Kết luận (5 điểm)

Khi nào nên dùng KG, khi nào Flat RAG là đủ?
1. **Khi nào Flat RAG là đủ:**
   - Khi câu hỏi thuộc dạng tìm kiếm sự thật đơn bước (**single-hop**) mà toàn bộ câu trả lời nằm trọn vẹn trong một đoạn văn bản cục bộ (như câu Q1 về định nghĩa tiền chất hay câu Q2 về danh sách bị cáo tử hình).
   - Với những tác vụ này, Flat RAG đạt điểm tuyệt đối (**Recall 1.00, Judge 2/2**) với chi phí rẻ hơn **4.15 lần** ($0.00013 vs $0.00054) và thời gian phản hồi nhanh hơn, không tốn thêm chi phí dựng đồ thị $0.00933 ban đầu.
2. **Khi nào BẮT BUỘC dùng Knowledge Graph (GraphRAG):**
   - Khi câu hỏi yêu cầu liên kết thông tin phân mảnh nằm ở các cơ sở tri thức khác nhau (**cross-KB multi-hop**) mà không một đoạn văn bản đơn lẻ nào chứa đủ thông tin (như câu Q3 nối mức án của Lê Minh Thành từ tin tức sang Điều 251 BLHS, hoặc câu Q5 suy luận từ 9.6kg MDMA sang khoản 4 Điều 250).
   - Trên các câu hỏi này, Flat RAG hoàn toàn thất bại (Q3 đạt Recall 0.00, Judge 0/2), trong khi GraphRAG đạt hiệu quả vượt trội (Q3 đạt **Recall 1.00, Judge 2/2**; nâng recall trung bình toàn bài từ 0.43 lên **0.63** và judge từ 1.00 lên **1.33**).
   - Đồ thị tri thức cũng giải quyết vượt trội các bài toán tổng hợp danh mục (**aggregation**) như câu Q6, cho phép gom cụm các vụ án và chỉ ra toàn bộ các điều luật liên đới.

---

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.19s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = openai:gpt-4o-mini | embedding = openai:text-embedding-3-small
[OK] KG-2 build_graph: 146 node / 289 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 13 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00064. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: **Cái Quang Huy** (vụ vận chuyển hơn 9,6kg MDMA và khoảng 406g Ketamine từ Đức về sân bay Nội Bài).

---

## Vấn đề gặp phải (không tính điểm)

Lỗi chưa giải quyết được: lệnh đã chạy, toàn bộ thông báo lỗi, những gì đã thử.
> - **Lỗi mã hóa terminal trên Windows:** Khi chạy `bench_kg.py` lần đầu gặp lỗi `UnicodeEncodeError: 'charmap' codec can't encode character '\u1eef'`.
>   *Cách xử lý:* Thiết lập biến môi trường `$env:PYTHONIOENCODING="utf-8"` trong PowerShell trước khi thực thi script Python.
> - **Cạnh tranh token giữa khoản 1 và khoản cao nhất:** Câu Q4 đòi hỏi mức án tối đa, nhưng cơ chế lọc chỉ ưu tiên khoản 1 và các khoản nhắc đến chất ma túy cụ thể, dẫn đến việc thiếu khoản 4 của Điều 255 BLHS trong ngữ cảnh. Đã phân tích chi tiết nguyên nhân và đề xuất phương án khắc phục tại mục Lỗi E2.
