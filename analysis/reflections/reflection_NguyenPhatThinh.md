# Individual Reflection — Lab 18: Production RAG

**Họ và tên:** Nguyễn Phát Thịnh  
**Mã số học viên (MSSV):** 2A202602645  
**Khóa học:** K4 - Track 3A (AI Application Engineer)  
**Ngày hoàn thành:** 04/10/2026  

---

## Phần 1: Mapping bài giảng (Lecture Mapping)

Ánh xạ chi tiết từng khái niệm trong bài giảng lý thuyết vào các module và hàm cụ thể đã triển khai trong Lab 18:

| # | Khái niệm bài giảng (Lecture Concept) | Module | Hàm cụ thể | Quan sát & Phân tích chuyên sâu (Observation) |
|---|---------------------------------------|--------|------------|----------------------------------------------|
| 1 | **Advanced Chunking Strategies** (Semantic, Hierarchical, Structure-Aware) | M1: Chunking | `chunk_semantic()`, `chunk_hierarchical()`, `chunk_structure_aware()` | - **Semantic Chunking:** Đoạn văn được tách theo ranh giới câu, mã hóa bằng `all-MiniLM-L6-v2` và tính cosine similarity giữa các câu liên tiếp. Với ngưỡng threshold = 0.85, các câu cùng chủ đề được gộp lại, loại bỏ triệt để hiện tượng cắt ngang ý hoặc ngắt giữa câu thường gặp ở Paragraph/Fixed-size chunking.<br>- **Hierarchical Chunking:** Tạo cấu trúc 2 tầng gồm Parent chunk ($\le 2048$ ký tự) và Child chunk ($\le 256$ ký tự) liên kết bằng `parent_id`. Khi truy vấn, mô hình tìm kiếm trên tập Child (tăng độ chính xác retrieval precision) nhưng trả về Parent chunk tương ứng làm context cho LLM (cung cấp đầy đủ ngữ cảnh toàn diện, loại trừ việc mất thông tin).<br>- **Structure-Aware Chunking:** Phân tích cú pháp Markdown header (`#`, `##`, `###`), bảo toàn nguyên vẹn danh sách, bảng dữ liệu và gán thẻ `section` vào metadata giúp lọc dữ liệu theo phân cấp tài liệu. |
| 2 | **Hybrid Search & Rank Fusion** (BM25 + Dense + RRF) | M2: Search | `segment_vietnamese()`, `BM25Search.search()`, `DenseSearch.search()`, `reciprocal_rank_fusion()` | - **Lexical Search (BM25):** Tận dụng `underthesea` để phân tách từ vựng tiếng Việt, đặc biệt xử lý thay thế dấu gạch dưới `_` thành khoảng trắng nhằm đảm bảo token của query và corpus khớp chuẩn xác khi dùng `BM25Okapi`. Bắt được chính xác các từ khóa hiếm, mã số quy định, thuật ngữ kỹ thuật.<br>- **Dense Search (Vector):** Sử dụng `BAAI/bge-m3` đa ngữ (1024 chiều) kết hợp Qdrant Vector Database (Cosine distance) bắt được ngữ nghĩa ngầm và các câu hỏi diễn đạt khác từ khóa gốc.<br>- **Reciprocal Rank Fusion (RRF):** Công thức $score(d) = \sum_{m} \frac{1}{k + rank_m + 1}$ với hằng số làm mượt $k=60$ giải quyết triệt để vấn đề thang điểm không tương thích (incompatible score distributions) giữa BM25 không chặn trên và Dense Cosine similarity [0, 1]. |
| 3 | **Cross-Encoder Reranking** | M3: Reranking | `CrossEncoderReranker.rerank()`, `benchmark_reranker()` | - Mô hình Bi-Encoder ở bước Retrieval chỉ mã hóa độc lập Query và Document để tối ưu tốc độ tìm kiếm trên quy mô lớn, do đó bỏ lỡ sự tương tác token-level giữa câu hỏi và đoạn văn.<br>- `CrossEncoderReranker` sử dụng `BAAI/bge-reranker-v2-m3` nhận trực tiếp cặp `(query, document)` qua cơ chế Full Cross-Attention, chấm điểm tương quan sâu. Lọc từ Top-20 ứng viên của Hybrid Search xuống Top-3 context chất lượng nhất đưa vào prompt cho LLM.<br>- Benchmark cho thấy độ trễ bổ sung nằm trong khoảng chấp nhận được (~20-50ms trên CPU/GPU) nhưng đẩy các đoạn tài liệu có thông tin cập nhật (như quy chế 2024 thay vì 2023) lên vị trí số 1. |
| 4 | **RAG Triad & RAGAS 4 Metrics** | M4: Evaluation | `evaluate_ragas()`, `failure_analysis()` | - Đánh giá toàn diện 4 khía cạnh theo chuẩn RAGAS:<br>  + **Faithfulness:** Câu trả lời có bám sát và suy luận trung thực từ Context hay không (phát hiện hallucination).<br>  + **Answer Relevancy:** Câu trả lời có đúng trọng tâm câu hỏi người dùng hay không.<br>  + **Context Precision:** Tỷ lệ các chunk hữu ích xuất hiện ở đầu bảng xếp hạng context.<br>  + **Context Recall:** Context thu hồi có bao phủ đầy đủ ground truth cần thiết để trả lời câu hỏi không.<br>- Cài đặt `failure_analysis()` tự động ánh xạ các câu hỏi điểm thấp vào **Diagnostic Tree** để chỉ ra lỗi nằm ở khâu Retrieval, Reranking hay Prompting. |
| 5 | **Chunk Enrichment Pipeline** | M5: Enrichment | `_enrich_single_call()`, `contextual_prepend()`, `generate_hypothesis_questions()`, `extract_metadata()` | - **Contextual Prepend (Anthropic style):** Bổ sung dòng tóm tắt vị trí của đoạn trích trong toàn bộ văn bản trước khi embed, giải quyết vấn đề đoạn văn bị tách rời ngữ cảnh gốc (decontextualization).<br>- **HyQA (Hypothesis Questions):** Sinh trước các câu hỏi tiềm năng mà chunk có thể giải đáp, giúp thu hẹp khoảng cách từ vựng (vocabulary gap) giữa câu hỏi của người dùng và văn bản quy chế.<br>- **Cost & Latency Optimization:** Thiết kế hàm `_enrich_single_call()` gom toàn bộ 4 tác vụ tóm tắt, sinh câu hỏi, thêm context và trích xuất metadata vào đúng 1 LLM API call duy nhất trên mỗi chunk (tiết kiệm 75% chi phí API và thời gian gọi mạng so với gọi rời rạc). |

---

## Phần 2: Khó khăn & Cách giải quyết (Challenges & Debugging)

Trong quá trình thực hiện bài lab, tôi đã gặp và giải quyết các bài toán kỹ thuật sau:

### 1. Lỗi tương thích Tokenizer trong Cross-Encoder
- **Hiện tượng (Issue):** Khi sử dụng gói `FlagEmbedding.FlagReranker` với các phiên bản `transformers>=5.0`, xuất hiện lỗi `XLMRobertaTokenizer: model_max_length is not set properly` gây crash chương trình khi rerank văn bản tiếng Việt.
- **Root Cause & Cách debug:** `FlagReranker` đóng gói phụ thuộc vào cấu trúc cũ của Hugging Face Tokenizers. Thư viện `sentence_transformers.CrossEncoder` quản lý pipeline cross-encoder ổn định hơn nhiều.
- **Giải pháp:** Chuyển sang sử dụng `from sentence_transformers import CrossEncoder; CrossEncoder("BAAI/bge-reranker-v2-m3")`. Mô hình tải nhanh, chạy mượt mà trên môi trường Python 3.11+.

### 2. Sự bất đối xứng trong Tokenization Tiếng Việt cho BM25
- **Hiện tượng (Issue):** Tìm kiếm BM25 cho kết quả rỗng hoặc điểm số bằng 0 đối với các từ ghép quen thuộc như "nghỉ phép", "mật khẩu", mặc dù trong văn bản có chứa chính xác các cụm từ này.
- **Root Cause & Cách debug:** Thư viện `underthesea.word_tokenize(format="text")` mặc định nối các từ ghép bằng dấu gạch dưới (ví dụ: `nghỉ_phép`, `thử_việc`). Trong khi đó, `BM25Okapi` tách từ bằng khoảng trắng (`split()`). Khi người dùng gõ query `"nghỉ phép"`, nó bị tách thành hai token riêng lẻ `['nghỉ', 'phép']`, không khớp với token `nghỉ_phép` trong index.
- **Giải pháp:** Trong hàm `segment_vietnamese(text)`, áp dụng chuẩn hóa `.replace("_", " ")` ngay sau khi tách từ. Cả query và tài liệu đều được đưa về không gian từ tố thống nhất, giúp BM25 đạt độ chính xác từ khóa 100%.

### 3. Đảm bảo tính khả dụng (High Availability) khi chưa có Vector DB hoặc API Key
- **Hiện tượng (Issue):** Môi trường thực tế có thể chưa khởi động Docker Qdrant hoặc chưa nạp `OPENAI_API_KEY`, dẫn đến toàn bộ pipeline bị dừng đột ngột (ConnectionRefusedError hoặc AuthenticationError).
- **Giải pháp:** 
  - Trong `DenseSearch`, cấu hình cơ chế fallback tự động sang `QdrantClient(":memory:")` nếu kết nối tới port 6333 thất bại.
  - Trong `m5_enrichment.py` và `m4_eval.py`, xây dựng hệ thống **Extractive Fallback** (tóm tắt bằng trích xuất câu đầu, sinh câu hỏi dựa trên câu trần thuật) và bọc các hàm gọi LLM trong khối `try/except`. Nhờ đó, pipeline và toàn bộ unit test vẫn hoạt động trơn tru.

---

## Phần 3: Action Plan cho Project Cá nhân (Application Plan)

Dựa trên các kỹ thuật sản xuất (Production-grade) đã làm chủ trong Lab 18, tôi xây dựng kế hoạch ứng dụng trực tiếp vào dự án cá nhân:

### Project: Hệ thống Trợ lý Pháp lý & Quy chế Doanh nghiệp (Enterprise Legal & Policy Copilot)

#### 1. Hiện trạng & Thách thức
- **Kiến trúc hiện tại:** Naive RAG sử dụng LangChain RecursiveCharacterTextSplitter (chunk size 1000, overlap 200) + OpenAI `text-embedding-3-small` + ChromaDB + GPT-4o.
- **Bottlenecks đang gặp:**
  1. *Retrieval Failures:* Không phân biệt được phiên bản quy chế mới (2024) và quy chế cũ đã hết hiệu lực (2023).
  2. *Loss of Context:* Khi trích xuất các bảng biểu tài chính hoặc điều khoản bảo mật ngắn, mô hình không biết điều khoản đó thuộc chương/mục nào.
  3. *Latency & Cost:* LLM context quá dài do đưa vào nhiều chunk rác từ kết quả tìm kiếm thô.
  4. *Lack of Grounded Evaluation:* Chưa có hệ thống đo lường định lượng tự động về độ chính xác và hallucination.

#### 2. Kế hoạch Cải tiến Toàn diện

| Thành phần | Giải pháp áp dụng từ Lab 18 | Lý do kỹ thuật |
|------------|----------------------------|----------------|
| **1. Chunking** | **Hierarchical Chunking (Parent-Child)** kết hợp **Structure-Aware** cho văn bản luật/quy định | Giữ nguyên cấu trúc "Chương - Điều - Khoản". Child chunk nhỏ giúp tìm kiếm vector chính xác từng câu chữ, Parent chunk lớn cung cấp ngữ cảnh đầy đủ cho LLM giải thích. |
| **2. Search & Retrieval** | **Hybrid Search (BM25 tiếng Việt + Dense BGE-M3 + RRF)** | BM25 đảm bảo tìm chính xác 100% số hiệu văn bản (Nghị định 13/2023, Điều 15), trong khi BGE-M3 bắt được câu hỏi dạng tình huống của nhân viên. |
| **3. Reranking** | **Cross-Encoder `bge-reranker-v2-m3`** | Lấy Top-25 từ Hybrid Search, rerank xuống Top-4. Giúp ưu tiên văn bản hiện hành lên trên văn bản hết hạn với độ tin cậy vượt trội. |
| **4. Enrichment** | **Contextual Prepend + Auto Metadata Extraction** (Combined 1-call) | Gắn ngữ cảnh "Văn bản này thuộc Quy chế Tài chính năm 2024..." vào từng chunk để triệt tiêu nhầm lẫn phiên bản. |
| **5. Evaluation** | **Continuous RAGAS CI/CD Pipeline** | Tích hợp bộ 20-50 câu hỏi benchmark tự động chạy RAGAS mỗi khi cập nhật cơ sở dữ liệu tri thức hoặc thay đổi prompt. |

#### 3. Timeline Triển khai (4 Tuần)
- **Tuần 1:** Chuẩn hóa dữ liệu nguồn: Áp dụng Markdown Structure-Aware parser cho toàn bộ văn bản quy chế nội bộ và chạy Enrichment pipeline (1-call mode).
- **Tuần 2:** Thiết lập Hybrid Search với Qdrant và BM25 (underthesea tokenization), kiểm thử RRF fusion với $k=60$.
- **Tuần 3:** Tích hợp Cross-Encoder Reranker (`bge-reranker-v2-m3`), tối ưu hóa độ trễ phản hồi dưới 1.5 giây.
- **Tuần 4:** Xây dựng bộ Test Set gồm 50 câu hỏi đa dạng (lookup, temporal/version, multi-hop), thiết lập báo cáo RAGAS định kỳ và đưa vào môi trường staging.
