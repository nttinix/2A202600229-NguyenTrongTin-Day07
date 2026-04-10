# Báo Cáo Lab 7: Embedding & Vector Store

**Họ tên:** Nguyễn Trọng Tín
**Nhóm:** 68
**Ngày:** 10/04/2026

---

## 1. Warm-up (5 điểm)

### Cosine Similarity (Ex 1.1)

**High cosine similarity nghĩa là gì?**
> Nó cho biết hai vector văn bản có hướng rất gần nhau trong không gian embedding. Về mặt ngữ nghĩa, điều này thường phản ánh hai câu hoặc hai đoạn có nội dung, chủ đề hoặc ý nghĩa tương tự nhau.

**Ví dụ HIGH similarity:**
- Sentence A: "The program highlights artificial intelligence and data science."
- Sentence B: "This course focuses on AI and data analysis techniques."
- Tại sao tương đồng: Cùng nói về một lĩnh vực chuyên môn với các thuật ngữ gần nghĩa.

**Ví dụ LOW similarity:**
- Sentence A: "Artificial intelligence is a branch of computer science."
- Sentence B: "Basketball is a popular sport played with a ball."
- Tại sao khác: Hai câu thuộc hai chủ đề hoàn toàn khác nhau, nên embedding sẽ có hướng xa nhau.

**Tại sao cosine similarity được ưu tiên hơn Euclidean distance cho text embeddings?**
> Cosine similarity tập trung vào hướng của vector nên phù hợp hơn để đo độ giống về ngữ nghĩa. Với text embeddings, độ dài câu có thể khác nhau nhưng nếu cùng chủ đề thì hướng vector vẫn gần nhau.

### Chunking Math (Ex 1.2)

**Document 10,000 ký tự, chunk_size=500, overlap=50. Bao nhiêu chunks?**
> *Trình bày phép tính:* step = 500 - 50 = 450. Số chunks = ceil((10000 - 500) / 450) + 1 = ceil(9500 / 450) + 1 = 22 + 1 = 23.
> *Đáp án:* 23 chunks.

**Nếu overlap tăng lên 100, chunk count thay đổi thế nào? Tại sao muốn overlap nhiều hơn?**
> Khi overlap = 100 thì step = 400, nên số chunks = ceil((10000 - 500) / 400) + 1 = ceil(23.75) + 1 = 25. Overlap lớn hơn giúp giữ ngữ cảnh tốt hơn ở ranh giới giữa các chunk, nhưng đổi lại số chunk tăng lên.

---

## 2. Document Selection — Nhóm (10 điểm)

### Domain & Lý Do Chọn

**Domain:** VinUni 20K AI Talent Program Handbook

**Tại sao nhóm chọn domain này?**
> Chương trình Đào tạo Nhân tài AI Thực chiến (20K AI) của VinUni là một domain thực tế, có cấu trúc tài liệu rõ ràng và nội dung đa dạng, từ lịch trình, học thuật đến chính sách. Đây là bộ dữ liệu phù hợp để kiểm thử khả năng retrieval và question answering của một hệ thống RAG.

### Data Inventory

| # | Tên tài liệu | Nguồn | Số ký tự | Metadata đã gán |
|---|--------------|-------|----------|-----------------|
| 1 | 01_highlights.md | VinUni AI Handbook | ~1500 | category: overview |
| 2 | 02_structure.md | VinUni AI Handbook | ~1000 | category: academic |
| 3 | 03_internships.md | VinUni AI Handbook | ~800 | category: logistics |
| 4 | 04_schedule.md | VinUni AI Handbook | ~2000 | category: logistics |
| 5 | 05_evaluation.md | VinUni AI Handbook | ~500 | category: academic |
| 6 | 06_lms.md | VinUni AI Handbook | ~800 | category: systems |
| 7 | 07_services.md | VinUni AI Handbook | ~2500 | category: logistics |
| 8 | 08_regulations.md | VinUni AI Handbook | ~2000 | category: policy |
| 9 | 09_completion_stipend.md | VinUni AI Handbook | ~2000 | category: policy |
| 10 | 10_faqs.md | VinUni AI Handbook | ~5000 | category: faq |

### Metadata Schema

| Trường metadata | Kiểu | Ví dụ giá trị | Tại sao hữu ích cho retrieval? |
|----------------|------|---------------|-------------------------------|
| category | string | academic, policy, logistics | Giúp lọc nhanh theo chủ đề trước khi search. |
| source | string | 20K AI Handbook v1.0 | Cho phép truy xuất lại nguồn tài liệu gốc khi cần kiểm chứng. |
| language | string | Vietnamese | Hữu ích nếu hệ thống cần hỗ trợ nhiều ngôn ngữ. |
| last_updated | string | 2025-04 | Giúp ưu tiên thông tin mới hơn khi tài liệu được cập nhật. |

---

## 3. Chunking Strategy — Cá nhân chọn, nhóm so sánh (15 điểm)

### Baseline Analysis

Chạy `ChunkingStrategyComparator().compare()` trên 3 tài liệu:

| Tài liệu | Strategy | Chunk Count | Avg Length | Preserves Context? |
|-----------|----------|-------------|------------|-------------------|
| 04_schedule.md | FixedSizeChunker (`fixed_size`) | 8 | 184.2 | Medium |
| 04_schedule.md | SentenceChunker (`by_sentences`) | 2 | 662.5 | High |
| 04_schedule.md | RecursiveChunker (`recursive`) | 9 | 144.3 | High |
| 08_regulations.md | FixedSizeChunker (`fixed_size`) | 14 | 187.6 | Medium |
| 08_regulations.md | SentenceChunker (`by_sentences`) | 6 | 390.8 | High |
| 08_regulations.md | RecursiveChunker (`recursive`) | 18 | 129.3 | High |
| 10_faqs.md | FixedSizeChunker (`fixed_size`) | 33 | 195.1 | Low |
| 10_faqs.md | SentenceChunker (`by_sentences`) | 12 | 479.1 | High |
| 10_faqs.md | RecursiveChunker (`recursive`) | 39 | 145.1 | High |

### Strategy Của Tôi

**Loại:** `SentenceChunker`

**Mô tả cách hoạt động:**
> Strategy này dùng regex `(?<=[.!?])(?:\s+|\n)` để tách văn bản thành các câu dựa trên dấu chấm, chấm hỏi, chấm than và khoảng trắng hoặc xuống dòng theo sau. Sau khi tách, các câu rỗng sẽ bị loại bỏ và mỗi nhóm tối đa 3 câu được ghép lại thành một chunk. Cách này giúp mỗi chunk vẫn giữ được đơn vị ngữ nghĩa tương đối hoàn chỉnh thay vì cắt cứng theo số ký tự.

**Tại sao tôi chọn strategy này cho domain nhóm?**
> Handbook của nhóm chủ yếu trình bày thông tin theo từng đoạn mô tả ngắn hoặc từng mục FAQ, nên ý nghĩa thường trọn vẹn trong vài câu liên tiếp. SentenceChunker tận dụng đúng pattern này để giữ ngữ cảnh tốt hơn so với fixed-size chunking.

**Code snippet (nếu custom):**
```python
# Không áp dụng vì tôi sử dụng SentenceChunker có sẵn trong package src.
```

### So Sánh: Strategy của tôi vs Baseline

| Tài liệu | Strategy | Chunk Count | Avg Length | Retrieval Quality? |
|-----------|----------|-------------|------------|--------------------|
| 10_faqs.md | FixedSizeChunker (best baseline) | 33 | 195.1 | Medium |
| 10_faqs.md | **SentenceChunker** | 12 | 479.1 | **High** |

### So Sánh Với Thành Viên Khác

| Thành viên | Strategy | Retrieval Score (/10) | Điểm mạnh | Điểm yếu |
|-----------|----------|----------------------|-----------|----------|
| Tôi | SentenceChunker | 6 | Giữ được ngữ nghĩa theo câu, chunk dễ đọc và coherent. | Chunk khá dài ở các tài liệu nhiều câu nối tiếp nên có thể làm retrieval kém chính xác hơn. |
| [Thành viên 1] | Header-based chunking | 8 | Tận dụng tốt cấu trúc phân cấp của handbook. | Phụ thuộc mạnh vào việc tài liệu có heading rõ ràng. |
| [Thành viên 3] | RecursiveChunker | 7 | Linh hoạt với nhiều loại văn bản, ít bị cắt giữa ý lớn. | Dễ sinh nhiều chunk nhỏ nếu văn bản có nhiều dấu ngắt. |

**Strategy nào tốt nhất cho domain này? Tại sao?**
> Với domain handbook có nhiều tiêu đề và mục nhỏ rõ ràng, header-based chunking là chiến lược phù hợp nhất ở mức tổng thể. Tuy vậy, SentenceChunker vẫn là một lựa chọn tốt cho các phần FAQ vì giữ được ngữ cảnh câu khá tự nhiên.

---

## 4. My Approach — Cá nhân (10 điểm)

Giải thích cách tiếp cận của bạn khi implement các phần chính trong package `src`.

### Chunking Functions

**`SentenceChunker.chunk`** — approach:
> Tôi dùng regex `(?<=[.!?])(?:\s+|\n)` để tách câu dựa trên dấu câu kết thúc và khoảng trắng hoặc ký tự xuống dòng phía sau. Sau khi chuẩn hóa bằng `strip()`, các câu được gom lại theo lô với `max_sentences_per_chunk = 3` để vừa giữ ngữ cảnh vừa đảm bảo số chunk không quá lớn.

**`RecursiveChunker.chunk` / `_split`** — approach:
> Thuật toán đệ quy thử tách văn bản theo thứ tự ưu tiên `\n\n`, `\n`, `. `, ` ` rồi cuối cùng mới fallback về cắt cứng theo kích thước. Base case là khi đoạn hiện tại rỗng, hoặc đã nhỏ hơn `chunk_size`, hoặc không còn separator nào để dùng. Cách này giúp giữ các đơn vị văn bản lớn nhất có thể trước khi phải chia nhỏ hơn.

### EmbeddingStore

**`add_documents` + `search`** — approach:
> Mỗi document được biến thành một record gồm `id`, `content`, `metadata` và `embedding`, sau đó lưu trong bộ nhớ và có thể đồng bộ sang ChromaDB nếu thư viện khả dụng. Khi search, hệ thống embed câu hỏi rồi tính điểm tương đồng bằng dot product giữa query embedding và embedding của từng record, sau đó sắp xếp giảm dần theo score.

**`search_with_filter` + `delete_document`** — approach:
> Với `search_with_filter`, tôi lọc metadata trước để thu hẹp tập ứng viên rồi mới tính điểm tìm kiếm, nhờ đó vừa nhanh hơn vừa đúng ngữ cảnh hơn. Với `delete_document`, tôi xóa tất cả chunk có `doc_id` tương ứng trong `_store`, đồng thời thử xóa cùng các `id` đó ở ChromaDB nếu đang dùng backend này.

### KnowledgeBaseAgent

**`answer`** — approach:
> Hàm `answer` trước hết gọi `store.search()` để lấy top-k chunk liên quan nhất, sau đó ghép chúng thành một khối context có kèm chỉ số chunk và score. Prompt được xây dựng theo hướng bắt buộc LLM chỉ dùng thông tin từ context đã truy xuất; nếu thiếu dữ liệu thì phải nói rõ là chưa đủ context.

### Test Results

```text
============================= test session starts =============================
platform win32 -- Python 3.12.6, pytest-8.3.4
collected 42 items

tests/test_solution.py::... PASSED

======================= 42 passed, 2 warnings in 0.09s =======================
```

**Số tests pass:** 42 / 42

---

## 5. Similarity Predictions — Cá nhân (5 điểm)

| Pair | Sentence A | Sentence B | Dự đoán | Actual Score | Đúng? |
|------|-----------|-----------|---------|--------------|-------|
| 1 | The cat sits on the mat. | The cat is sitting on the mat. | high | 0.98 | Đúng |
| 2 | Python is a programming language. | I like to eat apples. | low | 0.07 | Đúng |
| 3 | The weather is beautiful today. | It is a sunny day. | high | 0.71 | Đúng |
| 4 | The stock market is volatile. | Financial markets experience fluctuations. | high | 0.66 | Đúng |
| 5 | A dog is a loyal friend. | Computers process data quickly. | low | -0.05 | Đúng |

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn nghĩa?**
> Cặp 4 gây ấn tượng nhất vì hai câu không dùng cùng từ vựng nhưng vẫn đạt điểm tương đồng khá cao. Điều đó cho thấy embeddings không chỉ nhìn vào bề mặt từ ngữ mà còn nắm được quan hệ ngữ nghĩa giữa các khái niệm gần nhau.

---

## 6. Results — Cá nhân (10 điểm)

Chạy 5 benchmark queries của nhóm trên implementation cá nhân của bạn trong package `src`. **5 queries phải trùng với các thành viên cùng nhóm.**

### Benchmark Queries & Gold Answers (nhóm thống nhất)

| # | Query | Gold Answer |
|---|-------|-------------|
| 1 | Quy trình đánh giá học viên trong chương trình 20K AI như thế nào? | Được đánh giá dựa trên: 1. Lab hàng ngày; 2. Dự án #build; 3. Thi giữa kỳ (tuần 6); 4. Thực chiến doanh nghiệp; 5. Chuyên cần. |
| 2 | Học viên nhận được bao nhiêu tiền trợ cấp mỗi tháng và điều kiện nhận là gì? | Nhận 8 triệu VNĐ/tháng. Điều kiện: tuân thủ cam kết, duy trì chuyên cần, hoàn thành bài tập, tuân thủ kỷ luật và bảo mật. |
| 3 | Có những dịch vụ ăn uống và tiện ích nào tại campus VinUni? | Wifi riêng, thư viện, canteen, Highlands Coffee, Fresh Garden, trạm sạc xe điện miễn phí. |
| 4 | Cấu trúc chương trình đào tạo kéo dài mấy tuần và chia làm mấy giai đoạn? | 12 tuần, 3 giai đoạn: giai đoạn 1 (3 tuần) nền tảng, giai đoạn 2 (3 tuần) chuyên sâu, giai đoạn 3 (6 tuần) thực chiến. |
| 5 | Các quy định về chuyên cần trong giai đoạn học tập tại trường là gì? | Nghỉ tối đa 4 buổi trong giai đoạn 1 và 2. Không nghỉ 2 buổi liên tiếp trong 1 tuần. Cần báo trước nếu có lý do chính đáng. |

### Kết Quả Của Tôi

| # | Query | Top-1 Retrieved Chunk (tóm tắt) | Score | Relevant? | Agent Answer (tóm tắt) |
|---|-------|--------------------------------|-------|-----------|------------------------|
| 1 | Quy trình đánh giá học viên... | "Chương trình không đảm bảo chắc chắn..." | 0.74 | No | [MOCK] Sai về chủ đề. |
| 2 | Học viên nhận được bao nhiêu... | "Học viên được miễn học phí... 8 triệu VND/tháng" | 0.72 | Yes | [MOCK] Đáp ứng đúng thông tin trợ cấp. |
| 3 | Có những dịch vụ ăn uống... | "Hồ Chí Minh. Điều này nhằm đảm bảo học viên..." | 0.50 | No | [MOCK] Sai lệch kết quả retrieval. |
| 4 | Cấu trúc chương trình... | "# Quy định đào tạo  ###### **1. Quy định chuyên cần**..." | 0.79 | No | [MOCK] Nhầm sang phần chuyên cần. |
| 5 | Các quy định về chuyên cần... | "Học viên sẽ được phân vào các nhóm/track..." | 0.72 | No | [MOCK] Nhầm sang phần phân track. |

**Bao nhiêu queries trả về chunk relevant trong top-3?** 1 / 5

---

## 7. What I Learned (5 điểm — Demo)

**Điều hay nhất tôi học được từ thành viên khác trong nhóm:**
> Tôi học được rằng chunking theo header rất hiệu quả với các tài liệu có cấu trúc phân cấp rõ ràng như handbook hay policy guide. Khi tách theo heading, mỗi chunk thường bám sát một chủ đề nên retrieval chính xác hơn.

**Điều hay nhất tôi học được từ nhóm khác (qua demo):**
> Tôi nhận ra prompt engineering cho agent quan trọng không kém vector store. Cùng một tập chunk nhưng prompt rõ ràng hơn có thể giúp mô hình trả lời đúng phạm vi và giảm hallucination.

**Nếu làm lại, tôi sẽ thay đổi gì trong data strategy?**
> Tôi sẽ kết hợp thêm metadata phong phú hơn như section, heading và document priority để hỗ trợ filter tốt hơn. Ngoài ra, tôi muốn thử chiến lược hybrid giữa header-based chunking và sentence chunking cho các phần FAQ.

---

## Tự Đánh Giá

| Tiêu chí | Loại | Điểm tự đánh giá |
|----------|------|-------------------|
| Warm-up | Cá nhân | 5 / 5 |
| Document selection | Nhóm | 10 / 10 |
| Chunking strategy | Nhóm | 15 / 15 |
| My approach | Cá nhân | 10 / 10 |
| Similarity predictions | Cá nhân | 5 / 5 |
| Results | Cá nhân | 9 / 10 |
| Core implementation (tests) | Cá nhân | 30 / 30 |
| Demo | Nhóm | 5 / 5 |
| **Tổng** | | **94 / 100** |
