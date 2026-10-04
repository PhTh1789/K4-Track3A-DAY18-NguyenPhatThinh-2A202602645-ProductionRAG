# Failure Analysis — Lab 18: Production RAG

**Họ và tên học viên:** Nguyễn Phát Thịnh  
**Mã số học viên (MSSV):** 2A202602645  
**Khóa:** K4 - Track 3A  

---

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ |
|--------|---------------|------------|---|
| Faithfulness | 0.8933 | 0.7750 | -0.1183 |
| Answer Relevancy | 0.7642 | 0.7841 | +0.0199 |
| Context Precision | 0.9250 | 0.9042 | -0.0208 |
| Context Recall | 0.9250 | 0.8667 | -0.0583 |

> **Nhận xét tổng quan:**
> - Điểm số của Production RAG đạt chuẩn toàn diện với cả 4 metrics đều trên **0.75** (Faithfulness: 0.7750, Answer Relevancy: 0.7841, Context Precision: 0.9042, Context Recall: 0.8667), thỏa mãn điều kiện nhận **Bonus (+3 điểm)** theo Rubric.
> - `Answer Relevancy` tăng trưởng dương (+0.0199) nhờ việc áp dụng Cross-Encoder Reranking (`bge-reranker-v2-m3`) giúp chọn lọc chính xác các chunk liên quan trực tiếp đến trọng tâm câu hỏi.
> - `Faithfulness` và `Context Recall` của Production giảm nhẹ chủ yếu xuất phát từ 2 câu hỏi dạng Multi-hop (cần ghép 2 tài liệu độc lập) và quy tắc Prompt nghiêm ngặt `"Nếu không có → nói 'Không tìm thấy.'"`, khiến LLM từ chối trả lời thay vì suy đoán khi một phần context bị thiếu.

---

## Bottom-5 Failures (Diagnostic Tree Analysis)

### #1
- **Question:** Nghỉ phép không lương 20 ngày cần ai phê duyệt?
- **Expected (Ground Truth):** Nghỉ 16-30 ngày cần phê duyệt của Giám đốc điều hành (CEO). Lưu ý: nghỉ trên 14 ngày không lương, nhân viên phải tự đóng phần bảo hiểm của mình.
- **Got (LLM Answer):** Không tìm thấy.
- **Worst metric:** Faithfulness (0.0000) | Avg Score: 0.3750
- **Diagnostic Error Tree:** 
  - Output đúng? → Không. LLM từ chối trả lời ("Không tìm thấy.").
  - Context đúng? → Không. Trong Top-3 context sau Rerank không chứa đầy đủ đoạn quy định mức nghỉ 16-30 ngày trong file `nghi_phep_khong_luong.md`.
  - Query Rewrite / Search OK? → Chưa tối ưu. Query "20 ngày" là con số cụ thể nằm trong khoảng "16-30 ngày", BM25 không khớp được con số 20 với chuỗi "16-30", còn Dense Search bị nhiễu bởi các tài liệu nghỉ phép khác (nghỉ ốm, nghỉ phép năm).
  - Vị trí phát sinh lỗi: **M2 Search (BM25 & Dense retrieval)** và **M1 Chunking**.
- **Root cause:** Khoảng số liệu (range "16-30 ngày") trong văn bản không khớp từ khóa với số cụ thể "20 ngày" trong câu hỏi. Retrieval đưa vào context các chunk về nghỉ phép năm thay vì nghỉ không lương dài ngày.
- **Suggested fix:** Áp dụng kỹ thuật Numeric / Range Expansion trong Query Rewriting (hoặc trong bước Enrichment M5, trích xuất metadata khoảng số ngày được phép nghỉ).

### #2
- **Question:** Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?
- **Expected (Ground Truth):** Theo chính sách v2024: 15 ngày cơ bản + 3 ngày thâm niên (9÷3=3) = 18 ngày phép. Lương Senior (P3-P4): 20-35 triệu VNĐ/tháng.
- **Got (LLM Answer):** Không tìm thấy.
- **Worst metric:** Faithfulness (0.0000) | Avg Score: 0.3750
- **Diagnostic Error Tree:** 
  - Output đúng? → Không. LLM trả về "Không tìm thấy.".
  - Context đúng? → Không. Câu hỏi là dạng Multi-hop yêu cầu kết hợp 2 nguồn tài liệu riêng biệt: `nghi_phep_nam_v2024.md` (chính sách ngày phép thâm niên) và `bang_luong_2024.md` (khung lương Senior). Top-3 context sau khi Rerank chỉ giữ lại các chunk về nghỉ phép và loại mất chunk bảng lương.
  - Query Rewrite / Search OK? → Không. Câu hỏi chứa hai ý hỏi độc lập trong một câu, tìm kiếm đơn luồng làm loãng score.
  - Vị trí phát sinh lỗi: **M3 Reranking** (chỉ giữ Top-3) và **Pipeline Retrieval** (thiếu Query Decomposition).
- **Root cause:** Bottleneck Top-K ($K=3$) của Reranker quá hẹp đối với câu hỏi đa thực thể (multi-hop), dẫn đến việc loại bỏ các tài liệu liên quan đến vế thứ hai (bảng lương).
- **Suggested fix:** Nâng `RERANK_TOP_K` từ 3 lên 5 hoặc tách câu hỏi phức hợp thành 2 sub-queries trước khi retrieve (Query Decomposition).

### #3
- **Question:** Nếu cần mua một chiếc laptop 30 triệu cho nhân viên mới, ai phê duyệt và cần gì từ phòng CNTT?
- **Expected (Ground Truth):** Laptop 30 triệu nằm trong khoảng 5-50 triệu nên cần Giám đốc phòng ban (Director) phê duyệt. Ngoài ra, mua sắm thiết bị CNTT cần có xác nhận cấu hình kỹ thuật từ phòng CNTT trước khi đề xuất. Cần đính kèm ít nhất 3 báo giá vì trên 10 triệu.
- **Got (LLM Answer):** Để mua một chiếc laptop 30 triệu cho nhân viên mới, cần có xác nhận của phòng CNTT về cấu hình kỹ thuật trước khi đề xuất. Phê duyệt sẽ cần từ Giám đốc phòng ban vì số tiền vượt quá 5.000.000 VNĐ.
- **Worst metric:** Faithfulness (0.2500) | Avg Score: 0.6216
- **Diagnostic Error Tree:** 
  - Output đúng? → Đúng một phần. Trả lời đúng thẩm quyền phê duyệt (Director) và xác nhận CNTT, nhưng thiếu vế "cần đính kèm ít nhất 3 báo giá cạnh tranh vì trên 10 triệu".
  - Context đúng? → Thiếu một phần. Context được chọn tập trung vào thẩm quyền mua sắm CNTT, đoạn trích về thủ tục 3 báo giá cạnh tranh nằm ở mục khác trong `mua_sam.md` bị cắt rời.
  - Query Rewrite / Search OK? → Tương đối tốt.
  - Vị trí phát sinh lỗi: **M1 Chunking** (Hierarchical chunking cắt rời mục thủ tục báo giá khỏi mục thẩm quyền phê duyệt).
- **Root cause:** Đoạn văn bản quy định thủ tục báo giá và đoạn quy định thẩm quyền phê duyệt nằm ở hai section khác nhau, dẫn đến context bị phân mảnh.
- **Suggested fix:** Tăng kích thước `child_size` hoặc khi retrieve một child chunk từ `mua_sam.md`, đưa toàn bộ Parent Section hoặc tài liệu mua sắm hoàn chỉnh vào prompt.

### #4
- **Question:** Nhân viên tạm ứng 15 triệu, sau 20 ngày mới thanh toán. Bị phạt bao nhiêu?
- **Expected (Ground Truth):** Thời hạn thanh toán là 15 ngày. Quá hạn 5 ngày, bị tính phí 2%/tháng trên 15.000.000 VNĐ = 300.000 VNĐ/tháng (tính pro-rata khoảng 50.000 VNĐ cho 5 ngày).
- **Got (LLM Answer):** Nhân viên sẽ bị phạt 2% trên số tiền tạm ứng chưa hoàn ứng là 15 triệu VNĐ. Tính phạt như sau:\n\n15.000.000 VNĐ * 2% = 300.000 VNĐ.\n\nNhân viên sẽ bị phạt 300.000 VNĐ.
- **Worst metric:** Faithfulness (0.2500) | Avg Score: 0.7422
- **Diagnostic Error Tree:** 
  - Output đúng? → Sai về mặt tính toán số học. LLM tính nguyên tháng 300.000 VNĐ thay vì chia tỷ lệ pro-rata cho 5 ngày quá hạn (sau hạn 15 ngày).
  - Context đúng? → Có. Context có đầy đủ quy định về thời hạn hoàn ứng 15 ngày và mức phạt 2%/tháng.
  - Query Rewrite / Search OK? → Tốt.
  - Vị trí phát sinh lỗi: **Generation (LLM Reasoning)**.
- **Root cause:** Thiếu Chain-of-Thought hướng dẫn suy luận số học từng bước (step-by-step math reasoning), dẫn đến việc LLM áp dụng công thức nguyên tháng thay vì tính ngày trễ thực tế.
- **Suggested fix:** Cải thiện Prompt generation với chỉ dẫn: "Khi tính toán tiền phạt hoặc chi phí theo thời gian, hãy tính cụ thể số ngày quá hạn và quy đổi theo tỷ lệ ngày (pro-rata) nếu có".

### #5
- **Question:** Bao lâu phải đổi mật khẩu một lần?
- **Expected (Ground Truth):** Theo chính sách hiện hành (v2.0), mật khẩu phải được thay đổi mỗi 120 ngày. Chính sách cũ yêu cầu 90 ngày nhưng đã bị thay thế.
- **Got (LLM Answer):** Mật khẩu phải được thay đổi mỗi 120 ngày.
- **Worst metric:** Faithfulness (0.5000) | Avg Score: 0.7889
- **Diagnostic Error Tree:** 
  - Output đúng? → Đúng về mặt thực tế (120 ngày), nhưng RAGAS đánh giá 0.5 do câu trả lời ngắn gọn không nêu bối cảnh đối chiếu chính sách cũ v1.0 (90 ngày).
  - Context đúng? → Rất tốt. Reranker đã ưu tiên chính xác văn bản mới `mat_khau_v2.md` lên vị trí số 1.
  - Query Rewrite / Search OK? → Tốt.
  - Vị trí phát sinh lỗi: **Prompt Template / Generation**.
- **Root cause:** Prompt hiện tại yêu cầu "Trả lời CHỈ dựa trên context", khiến LLM chỉ trích xuất đúng con số hiện hành 120 ngày mà không diễn giải thêm sự thay đổi so với quy định cũ.
- **Suggested fix:** Thêm chỉ dẫn trong System Prompt: "Đối với các câu hỏi về chính sách có sự thay đổi giữa các phiên bản, hãy nêu rõ quy định hiện hành và ghi chú phiên bản cũ đã hết hiệu lực".

---

## Case Study (Phân tích chuyên sâu)

**Question chọn phân tích:**  
`Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?`

### 1. Phân tích Bản chất Vấn đề
Đây là câu hỏi điển hình thuộc dạng **Multi-hop / Multi-entity Question**, đòi hỏi hệ thống RAG phải thu hồi và tổng hợp thông tin từ **hai nguồn tri thức độc lập**:
- Nguồn 1: [`data/nghi_phep_nam_v2024.md`](file:///d:/AI20K/Phase%202_Track3-Application_Lab/Lab/K4-Track3A-DAY18-NguyenPhatThinh-2A202602645-ProductionRAG/data/nghi_phep_nam_v2024.md) (Chính sách ngày phép: cơ bản 15 ngày + 1 ngày cho mỗi 3 năm thâm niên $\rightarrow$ 9 năm được cộng 3 ngày = 18 ngày).
- Nguồn 2: [`data/bang_luong_2024.md`](file:///d:/AI20K/Phase%202_Track3-Application_Lab/Lab/K4-Track3A-DAY18-NguyenPhatThinh-2A202602645-ProductionRAG/data/bang_luong_2024.md) (Bảng thang bảng lương chức danh: Senior tương ứng cấp bậc P3-P4 với dải lương 20 - 35 triệu VNĐ/tháng).

### 2. Error Tree Walkthrough
1. **Output đúng?** $\rightarrow$ **Không**. LLM phản hồi `"Không tìm thấy."`.
2. **Context đúng?** $\rightarrow$ **Không đầy đủ**.
   - Kiểm tra log pipeline, bước Hybrid Search (BM25 + Dense) lấy Top-20 candidates bao gồm cả các chunk về nghỉ phép và bảng lương.
   - Tuy nhiên, khi đi qua `CrossEncoderReranker` với tham số `top_k=3`, Cross-Encoder nhận query chứa cả hai vế nhưng tính điểm tương đồng cao hơn cho các chunk có từ vựng "thâm niên", "nghỉ phép", "ngày phép".
   - Kết quả: Toàn bộ Top-3 context được chọn đều thuộc về chủ đề nghỉ phép (`nghi_phep_nam_v2024.md`, `nghi_phep_dac_biet.md`, `nghi_phep_nam_v2023.md`). Không có chunk nào từ `bang_luong_2024.md` lọt vào Top-3!
3. **LLM xử lý:** Do prompt hệ thống có chỉ thị nghiêm ngặt: `"Trả lời CHỈ dựa trên context. Nếu không có → nói 'Không tìm thấy.'"` $\rightarrow$ Vì không tìm thấy thông tin lương, LLM từ chối trả lời toàn bộ câu hỏi để tránh bịa đặt thông tin (anti-hallucination guardrail).
4. **Vị trí phát sinh lỗi:** **M3 Reranking** (Top-K quá hẹp đối với câu hỏi Multi-hop) và **Query Single-stream**.

### 3. Giải pháp Khắc phục Triệt để
1. **Áp dụng Query Decomposition (Phân rã câu hỏi):**
   - Trước khi đưa vào Search, sử dụng một LLM call nhẹ để phân tích câu hỏi thành 2 sub-queries:
     - Sub-query 1: *"Nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm?"*
     - Sub-query 2: *"Mức lương của nhân viên Senior là bao nhiêu?"*
   - Chạy Hybrid Search độc lập cho từng sub-query, sau đó hợp nhất các context đại diện từ mỗi nhánh trước khi rerank.
2. **Mở rộng Rerank Top-K:**
   - Tăng `RERANK_TOP_K` từ **3** lên **5** hoặc **6**. Độ trễ context chỉ tăng khoảng 200 tokens nhưng đảm bảo bao phủ đầy đủ các thực thể trong câu hỏi phức hợp.
3. **Adaptive Prompting:**
   - Cho phép LLM trả lời phần thông tin đã tìm thấy và ghi chú rõ phần thông tin còn thiếu thay vì từ chối toàn bộ: `"Nếu thiếu một phần thông tin, hãy trả lời phần có trong context và thông báo phần chưa tìm thấy"`.

---

## Nếu có thêm 1 giờ, sẽ optimize:
1. **Query Decomposition & HyDE:** Tích hợp bộ tiền xử lý câu hỏi để tách câu hỏi multi-hop và tạo giả định câu trả lời nhằm nâng `Context Recall` lên $\ge 0.95$.
2. **Dynamic Top-K Reranking:** Tự động điều chỉnh `top_k` (từ 3 lên 5) nếu phát hiện câu hỏi chứa liên từ *"và"*, *"đồng thời"* hoặc có độ dài $> 15$ từ.
3. **Chain-of-Thought Numeric Prompting:** Tinh chỉnh prompt template để hỗ trợ các bài toán tính toán chi phí, khấu hao, thời gian thử việc và phạt tạm ứng pro-rata chuẩn xác.


