# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Nguyễn Hải Long  **MSSV:** 2A202602471  **Ngày:** 05/10/2026

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

---

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```text
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112     77.2
graph       196     91958     4805   0.00939    165.7

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       47   0.00013     1.71
graph       0.94   1.83     4738       74   0.00075     2.63
```

| Chỉ số | Flat | Graph | Graph / Flat |
| :--- | :--- | :--- | :--- |
| **Indexing USD** | $0.00112 | $0.00939 | $\times 8.38$ |
| **Indexing giây** | 77.2s | 165.7s | $\times 2.15$ |
| **Mỗi câu: USD** | $0.00013 | $0.00075 | $\times 5.77$ |
| **Mỗi câu: giây** | 1.71s | 2.63s | $\times 1.54$ |
| **Mỗi câu: in_tok** | 694 | 4738 | $\times 6.83$ |

**Chi phí tăng thêm đến từ đâu?**
> Ở pha Indexing, chi phí GraphRAG tăng chủ yếu do việc gọi LLM tuần tự để trích xuất có cấu trúc (JSON mode) cho 20 bài báo tin tức (tốn thêm 20 cuộc gọi LLM với hơn 35.000 input tokens và 4.805 output tokens). Ở pha Querying, chi phí tăng gấp ~5.77 lần và input tokens tăng ~6.83 lần do prompt của GraphRAG không chỉ chứa các đoạn text chunks mà còn được làm giàu bởi hàng chục dữ kiện quan hệ đa bước (facts) lấy từ đồ thị Neo4j.

---

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **Q1** | `single-hop-law` | 1.00 / 2 | 1.00 / 2 | **Hòa** | Khái niệm tiền chất nằm trọn vẹn trong một đoạn văn bản của Luật PCMT 2021 nên cả 2 bên đều lấy đủ thông tin. |
| **Q2** | `single-hop-news` | 1.00 / 2 | 1.00 / 2 | **Hòa** | Tên 2 bị cáo tử hình nằm tập trung trong 1 bài báo về vụ 36kg ma túy nên vector search đã đủ để trả lời chính xác. |
| **Q3** | `cross-kb` | 0.00 / 0 | 1.00 / 2 | **Graph** | Flat RAG không nối được tên người ở tin tức sang Điều luật hình sự nên báo "Không đủ thông tin", trong khi GraphRAG đi qua node `Crime` trả lời đủ 36 tháng tù và Điều 251 khoản 1. |
| **Q4** | `cross-kb` | 0.00 / 0 | 1.00 / 2 | **Graph** | Flat RAG báo thiếu thông tin; GraphRAG nhờ quan hệ `HAS_MAX_PENALTY` đã truy xuất trúng Điều 255 và mức án kịch khung là tù chung thân. |
| **Q5** | `cross-kb-multi-hop` | 0.60 / 1 | 1.00 / 2 | **Graph** | Flat RAG đoán sai thành "khoản b)", trong khi GraphRAG đối chiếu khối lượng 9.6kg MDMA xác định đúng Khoản 4 Điều 250 và mức án tử hình. |
| **Q6** | `aggregation` | 0.00 / 1 | 0.67 / 1 | **Graph** | Flat RAG chỉ tìm được thông tin cục bộ; GraphRAG nhờ liên kết chất MDMA đã gom đủ cả 3 vụ án (Cái Quang Huy, ba thanh niên, Viện Pháp y tâm thần). |

---

## 3. Phân tích lỗi (20 điểm)

### Lỗi E2: Thiếu ngữ cảnh luật (Bỏ sót khung hình phạt tối đa)

- **Hiện tượng:** Khi người dùng hỏi về mức hình phạt cao nhất của một hành vi (như câu Q4 hỏi mức án tối đa của giang hồ Hoàng Nato về hành vi tổ chức sử dụng ma túy), hệ thống dễ bỏ sót khung phạt tối đa nếu chỉ lọc theo loại chất ma túy.
- **Bằng chứng:** Trong `ket_qua_benchmark_kg.txt`, Flat RAG hoàn toàn thất bại (Judge = 0, trả lời *"Không đủ thông tin"*). Ở bản thiết kế ontology gợi ý ban đầu, Cypher chỉ lọc các khoản có nhắc tên chất ma túy mà vụ án dính líu, nhưng Khoản 4 của Điều 255 BLHS quy định: *"Phạm tội thuộc một trong các trường hợp sau đây, thì bị phạt tù 20 năm hoặc tù chung thân: ... làm chết 02 người trở lên..."* — khoản này **không hề chứa tên chất ma túy nào**. Hậu quả là gợi ý chỉ bốc được Khoản 1 (2 - 7 năm tù) và mất từ khóa `"chung thân"`.
- **Cypher chứng minh trên Neo4j:**

```cypher
MATCH (a:Article {id: 'Điều 255 BLHS'})-[:HAS_MAX_PENALTY]->(cl:Clause)
RETURN cl.number, cl.penalty;
```

```text
number: 4
penalty: "phạt tù 20 năm hoặc tù chung thân"
```

- **Nguyên nhân:** Các tình tiết định khung tăng nặng kịch khung trong BLHS Việt Nam thường dựa trên hậu quả đặc biệt nguy hiểm (chết nhiều người, có tổ chức, tái phạm nguy hiểm) chứ không liệt kê lại tên chất ma túy.
- **Đề xuất sửa:** Thiết kế thêm quan hệ trực tiếp `[:HAS_MAX_PENALTY]` nối từ `Article` tới `Clause` có khung hình phạt tù cao nhất, đồng thời trong logic truy vấn `context()` kiểm tra từ khóa `"tối đa"`, `"cao nhất"` để tự động kích hoạt duyệt theo cạnh này.

---

### Lỗi E3: Trùng thực thể và phân mảnh do từ lóng báo chí

- **Hiện tượng:** Báo chí thường dùng tên gọi thông tục, tiếng lóng (như *"thuốc lắc"*, *"ma túy đá"*, *"hàng đá"*, *"heroin"*) thay vì tên khoa học chuẩn tắc trong luật (`MDMA`, `Methamphetamine`, `Heroine`). Điều này khiến việc đếm và tổng hợp vụ án ở câu hỏi tổng hợp Q6 bị sót dữ liệu.
- **Bằng chứng:** Ở câu Q6, Flat RAG trả về Recall = 0.00 do các bài báo gọi chất bằng nhiều từ ngữ khác nhau. Ngay cả đồ thị ban đầu nếu chỉ tìm theo chuỗi cứng `MDMA` thì bài báo chỉ viết *"thuốc lắc"* sẽ không được nối vào node `MDMA`.
- **Cypher chứng minh trên Neo4j:**

```cypher
MATCH (s:Substance {name: 'MDMA'})-[:HAS_ALIAS]->(sa:SubstanceAlias)
RETURN s.name, collect(sa.name) AS aliases;
```

```text
s.name: "MDMA"
aliases: ["thuốc lắc", "ecstasy", "kẹo"]
```

- **Nguyên nhân:** Ngôn ngữ báo chí hướng tới đại chúng nên sử dụng từ ngữ đời thường, trong khi luật pháp bắt buộc dùng danh pháp khoa học. Nếu ontology xem `Substance` là một chuỗi văn bản đơn lẻ mà không có mô hình hóa bí danh (aliases), đồ thị sẽ bị đứt gãy liên kết ngữ nghĩa.
- **Đề xuất sửa:** Tạo thực thể `SubstanceAlias` và liên kết qua quan hệ `[:HAS_ALIAS]`, kết hợp cập nhật hàm `find_substances` quét cả danh mục từ đồng nghĩa để chuẩn hóa về node `Substance` gốc.

---

## 4. Kết luận (5 điểm)

- **Khi nào nên dùng Flat RAG:** Flat RAG là lựa chọn tối ưu khi khối lượng câu hỏi là dạng tìm kiếm sự thật đơn lẻ (single-hop), câu trả lời nằm gọn trong một đoạn văn bản (như định nghĩa ở Q1 hoặc danh sách bị cáo ở Q2). Flat RAG tiết kiệm chi phí gấp **5.77 lần** ($0.00013 so với $0.00075) và phản hồi nhanh hơn **1.54 lần** (1.71s so với 2.63s) mà vẫn đạt độ chính xác tối đa (Recall = 1.0, Judge = 2).
- **Khi nào bắt buộc phải dùng GraphRAG:** Khi hệ thống phải phục vụ các bài toán suy luận phức tạp xuyên nhiều tài liệu (cross-KB) và tổng hợp thông tin (aggregation). Số liệu thực nghiệm chứng minh: Ở các câu hỏi liên miền tri thức (Q3, Q4, Q5), Flat RAG gần như hoàn toàn bất lực (Recall rơi xuống 0.00 và 0.60, thường xuyên báo không đủ thông tin), trong khi GraphRAG đạt **Recall = 0.94** và **Judge = 1.83**. Chi phí đầu tư ban đầu cho việc dựng đồ thị tri thức (khoảng $0.009 USD) hoàn toàn xứng đáng với giá trị tri thức và độ tin cậy pháp lý vượt trội mà hệ thống mang lại.

---

## 5. Tự kiểm (5 điểm)

```text
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.17s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = openrouter:openai/gpt-4o-mini | embedding = openrouter:openai/text-embedding-3-small
[OK] KG-2 build_graph: 163 node / 325 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 22 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00077.
```

- Ảnh Neo4j đã lưu đủ trong thư mục:
  - [`report/img/kg_count.png`](img/kg_count.png)
  - [`report/img/kg_cross_kb.png`](img/kg_cross_kb.png)
  - [`report/img/kg_my_case.png`](img/kg_my_case.png)
- Người đã chọn cho `kg_my_case.png`: **Cái Quang Huy** (Vụ vận chuyển trái phép 9.6kg MDMA và Ketamine từ Đức về Việt Nam).

---

## Vấn đề gặp phải (không tính điểm)

- Ban đầu lệnh chạy trên Windows gặp lỗi mã hóa ký tự `UnicodeEncodeError: 'charmap'` khi in tiếng Việt ra terminal. Đã xử lý triệt để bằng cách thiết lập biến môi trường `$env:PYTHONIOENCODING="utf-8"`.
- Trên giao diện Neo4j Browser mới, câu lệnh `:clear` không thể viết chung một khung với câu truy vấn `MATCH`, cần chạy riêng hoặc xóa `:clear` trước khi thực thi truy vấn Cypher.
