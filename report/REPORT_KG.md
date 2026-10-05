# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Nguyễn Tú Tài  **MSSV:** 2A202602455  **Ngày:** 05/10/2026

> Số liệu trong báo cáo lấy nguyên từ `ket_qua_benchmark_kg.txt`, chạy với Gemini 3.5 Flash-Lite, Gemini Embedding, `top_k=3`, `chunk_size=800`.

## 1. Chi phí (10 điểm)

Hai bảng từ file kết quả:

```text
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176         0        0   0.00000    103.6
graph       196     34619     5378   0.02383    215.1
```

```text
== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.51   1.33      696       74   0.00039     1.58
graph       0.89   1.83     2862      125   0.00117     2.07
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | ---: | ---: | ---: |
| Indexing USD | 0.00000 | 0.02383 | Không xác định vì mẫu số bằng 0 |
| Indexing giây | 103.6 | 215.1 | 2.08× |
| Mỗi câu: USD | 0.00039 | 0.00117 | 3.00× |
| Mỗi câu: giây | 1.58 | 2.07 | 1.31× |
| Mỗi câu: in_tok | 696 | 2862 | 4.11× |

**Chi phí tăng thêm đến từ đâu?** Flat indexing chỉ có 176 lượt embedding; Graph indexing dùng lại 176 embedding đó và thêm 20 lượt chat để trích xuất 20 bài báo, vì vậy tăng lên 196 lượt gọi, 34.619 input token và 5.378 output token. Khi hỏi, GraphRAG đưa cả chunk lẫn các fact multi-hop vào prompt nên input token trung bình tăng 4,11 lần và USD tăng 3,00 lần. Gemini Embedding qua endpoint tương thích OpenAI không trả token usage, nên bảng ghi chi phí embedding của Flat bằng 0 và không thể tính tỉ lệ USD indexing hữu hạn.

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao |
| --- | --- | ---: | ---: | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Flat | Hai bên đúng đủ định nghĩa, nhưng Flat dùng prompt ngắn và rẻ hơn. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Flat | Một chunk tin đã chứa đủ hai tên bị cáo nên graph không tạo thêm lợi ích về độ đúng. |
| Q3 | cross-kb | 0.33 / 1 | 1.00 / 2 | Graph | Graph đi từ Lê Minh Thành → vụ án → tội danh → Điều 251 → khoản 1, còn Flat chỉ lấy được mức án và tội. |
| Q4 | cross-kb | 0.33 / 1 | 1.00 / 2 | Graph | Graph nối alias Hoàng Nato với Điều 255 và lấy mọi khoản để tìm mức cao nhất là tù chung thân. |
| Q5 | cross-kb-multi-hop | 0.40 / 1 | 1.00 / 2 | Graph | Graph kết hợp người, vụ án, MDMA hơn 9,6 kg và khoản 4 Điều 250 để suy ra đúng khung 20 năm/chung thân/tử hình. |
| Q6 | aggregation | 0.00 / 1 | 0.33 / 1 | Graph theo recall; judge hòa | Flat tách hai kiện hàng của Huy thành hai vụ và bỏ sót vụ Pháp y; Graph tìm đúng Pháp y nhưng không nêu đầy đủ tên Cái Quang Huy/Lê Minh Thành và trả thêm Hoàng Nato. |

Quy luật quan sát được: câu single-hop nằm trọn trong một tài liệu thì Flat đủ tốt và tiết kiệm hơn. Với Q3-Q5, khi câu hỏi phải nối tin tức với luật hoặc suy luận nhiều bước, Graph thắng rõ rệt cả recall lẫn judge; ở Q6 tổng hợp nhiều tài liệu, Graph chỉ tăng recall còn judge hòa.

## 3. Phân tích lỗi (20 điểm)

### Lỗi E3: Một vụ ngoài đời thành nhiều node `Case`

- **Hiện tượng:** Cái Quang Huy được nối tới hai node `Case` mô tả cùng vụ vận chuyển MDMA qua Nội Bài. Node đúng nối được tới Điều 250; node còn lại lấy từ đoạn gợi ý bài liên quan ở cuối một bài báo về Lê Minh Thành nên mang sai `doc_id` và không có cầu nối luật.
- **Bằng chứng:**

```cypher
MATCH (p:Person {name:'Cái Quang Huy'})-[:INVOLVED_IN]->(k:Case)
OPTIONAL MATCH (k)-[:CHARGED_WITH]->(c:Crime)<-[:DEFINES]-(a:Article)
RETURN k.name AS case_name, k.doc_id AS doc_id, k.source_title AS source_title,
       collect(DISTINCT c.name) AS crimes, collect(DISTINCT a.id) AS articles
ORDER BY doc_id;
```

```text
case_name: Vụ vận chuyển hơn 10kg ma túy từ Đức về Việt Nam qua sân bay Nội Bài
doc_id: news-100260917203001265
source_title: Từ mối quen biết tại nhà hàng ở Berlin đến những kiện hàng chứa hơn 10kg ma túy về Việt Nam
crimes: [vận chuyển trái phép chất ma túy]
articles: [Điều 250 BLHS]

case_name: Vụ vận chuyển ma túy qua sân bay Nội Bài
doc_id: news-100260918080821054
source_title: Góp 14 triệu đồng mua ma túy rồi nói đã ‘rút lui’, 3 thanh niên kháng cáo kêu oan
crimes: []
articles: []
```

- **Nguyên nhân:** Lỗi bắt đầu ở dữ liệu/crawler: cuối `news-100260918080821054.md` còn đoạn giới thiệu bài liên quan về Cái Quang Huy. LLM coi đoạn này là một vụ thứ hai. Ở bước dựng graph, `Case` lại `MERGE` theo trường `name` tự do do LLM sinh; hai cách đặt tên khác nhau không được gộp dù cùng người, địa điểm và sự kiện.
- **Đề xuất sửa:** cắt phần bài liên quan trước khi gửi prompt; thêm `case_key` ổn định từ tập người + ngày + địa điểm + tội danh; sau trích xuất chạy entity resolution để gộp các `Case` có cùng người/chất/khối lượng. Với case không có `CHARGED_WITH`, cần đánh dấu để rà soát thay vì đưa thẳng vào graph trả lời.

### Lỗi E4: `recall` và `judge` mâu thuẫn ở Q6 Flat

- **Hiện tượng:** Câu Q6 của Flat được judge chấm đúng một phần (`1`) nhưng keyword recall bằng `0.00`; hai phép đo khác nhau dù câu trả lời đã nhận ra một phần các vụ cần tìm.
- **Bằng chứng:** nguyên văn phần tương ứng trong file benchmark:

```text
--- Q6 [aggregation] flat recall=0.00 judge=1 1.71s
Dựa trên ngữ cảnh, tất cả 3 vụ việc đều có liên quan đến ma túy MDMA:

1. **Vụ việc [1]:** Lực lượng chức năng phát hiện bên trong thùng hàng có các viên nén màu xanh là ma túy MDMA (khối lượng gần 4,3kg) liên quan đến các nhân vật như Đạt và Huy.
2. **Vụ việc [2]:** Công an bắt quả tang Thành mang 5 viên nén ma túy MDMA (được gọi là ma túy "kẹo") đi bán theo yêu cầu của tài khoản "Cường Quốc".
3. **Vụ việc [3]:** Kết quả giám định xác định số viên nén hình tam giác màu hồng - xám trong kiện hàng do Huy gửi là ma túy MDMA (tổng khối lượng hơn 5,3kg) liên quan đến Đức.
```

`must_include` của Q6 là `Cái Quang Huy`, `Lê Minh Thành`, `Pháp y tâm thần`. Câu trả lời dùng tên rút gọn `Huy`, `Thành`, `Đức`, nên phép đo substring không tìm thấy chuỗi đầy đủ nào và cho recall bằng 0 dù đã nhận ra hai người đầu. Hai mục 1 và 3 thực chất là hai kiện hàng trong cùng vụ Cái Quang Huy, còn vụ Pháp y tâm thần bị bỏ sót, nên judge cho `1` phản ánh đúng hơn mức độ đúng một phần.

- **Nguyên nhân:** `keyword_recall` chỉ đo sự xuất hiện nguyên văn, không hiểu alias/tên rút gọn hay tương đương ngữ nghĩa. Ngược lại, LLM judge thấy đủ ba gạch đầu dòng về MDMA nhưng không kiểm tra định danh sự kiện, nên không nhận ra hai gạch đầu dòng thuộc cùng một vụ và một vụ vàng đã bị thiếu.
- **Đề xuất sửa:** chuẩn hóa tên người và alias trước khi tính recall; chấm aggregation như một tập `case_id` bằng precision/recall/F1; yêu cầu judge xuất từng fact đúng/sai và trừ điểm cho phần tử thừa. Có thể dùng hai judge độc lập hoặc kiểm tra cấu trúc để giảm độ lệch của một LLM judge.

## 4. Kết luận (5 điểm)

Flat RAG phù hợp khi câu hỏi single-hop và đáp án nằm trong một chunk: Q1–Q2 hai pipeline đều đạt recall 1.00, judge 2, trong khi Flat rẻ và nhanh hơn. KG đáng chi phí khi cần nối hai KB hoặc suy luận nhiều bước: ở Q3–Q5, recall tăng từ 0.33/0.33/0.40 lên 1.00 và judge tăng từ 1 lên 2. Với aggregation Q6, Graph chỉ tăng recall từ 0.00 lên 0.33 và judge vẫn bằng 1, cho thấy graph không tự sửa được lỗi trùng thực thể/thiếu tên từ bước trích xuất.

Đổi lại, Graph tốn khoảng 3,00 lần USD, 1,31 lần thời gian và 4,11 lần input token cho mỗi câu, cộng chi phí indexing một lần là 0,02383 USD và 215,1 giây. Vì vậy KG đáng tiền với hệ thống phải trả lời thường xuyên các câu cross-KB, cần đường dẫn giải thích hoặc tổng hợp nhiều vụ; Flat đủ cho tra cứu đơn tài liệu, lưu lượng thấp, hoặc khi độ trễ/chi phí quan trọng hơn suy luận quan hệ. Kết quả E3 và Q6 cho thấy KG chỉ đáng tin khi pipeline entity resolution và làm sạch nguồn đủ tốt.

## 5. Tự kiểm (5 điểm)

```text
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.08s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = gemini:gemini-3.5-flash-lite | embedding = gemini:gemini-embedding-001
[OK] KG-2 build_graph: 148 node / 294 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 6 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00255. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j:

- `report/img/kg_count.png`
- `report/img/kg_cross_kb.png`
- `report/img/kg_my_case.png`

Người đã chọn cho `kg_my_case.png`: **Cái Quang Huy**.

## Vấn đề gặp phải (không tính điểm)

Model Gemini mặc định trong đề (`gemini-2.5-flash-lite`) không còn cấp cho người dùng mới, nên đã cập nhật sang `gemini-3.5-flash-lite` theo thông báo của API và cập nhật bảng giá 0,30/2,50 USD trên một triệu input/output token. Free tier Gemini Embedding giới hạn 100 request/phút; `src/llm.py` được bổ sung retry theo đúng `retryDelay` của API, không giảm số chunk và không sửa `bench_kg.py`.
