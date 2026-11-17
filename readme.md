# BÁO CÁO BÀI TẬP: SO SÁNH TRADE-OFF CÁC GIẢI PHÁP RA LỆNH CHO KỸ THUẬT PROMPTING NÂNG CAO (LOGIC & CHUỖI)

---

## 1. Mục tiêu & Bối cảnh bài toán

* **Mục tiêu:** Rèn luyện tư duy phản biện thông qua việc thiết kế, đối chiếu hai phương án viết prompt khác nhau cho bài toán xử lý logic chuỗi phức tạp, từ đó lựa chọn giải pháp tối ưu nhất dựa trên sự đánh đổi (trade-off) về hiệu năng, chi phí và độ tin cậy.
* **Bối cảnh bài toán:** Xây dựng luồng xử lý tự động cho hệ thống Chăm sóc Khách hàng Thương mại điện tử. Hệ thống cần tiếp nhận một phản hồi khiếu nại dài và đa tầng của khách hàng, phân tích nguyên nhân (Chain-of-Thought), chia nhỏ thành các tác vụ xử lý theo chuỗi (Prompt Chaining: Trích xuất thông tin $\rightarrow$ Đối chiếu chính sách $\rightarrow$ Đề xuất phương án đền bù) và đảm bảo không bị đứt gãy dữ liệu (context loss) hay sinh ảo giác (hallucination) khi gặp dữ liệu đầu vào thiếu hụt.

---

## 2. Bước 1: Đề xuất 2 Prompt mẫu

### Giải pháp 1: Prompt Đơn giản, Ngắn gọn (Zero-shot / Basic CoT)
*Đặc điểm: Tập trung vào sự súc tích, giao quyền suy luận tự nhiên cho mô hình, ít rào cản kỹ thuật.*

```text
Bạn là nhân viên chăm sóc khách hàng. Hãy đọc khiếu nại sau của khách hàng, suy nghĩ từng bước (step-by-step) để tìm ra vấn đề, kiểm tra xem có hợp lệ để hoàn tiền không (chính sách: chỉ hoàn tiền nếu hàng lỗi do nhà sản xuất trong vòng 7 ngày), và viết email trả lời khách.

Khiếu nại của khách: "{{INPUT_DATA}}"
```

---

### Giải pháp 2: Prompt Chi tiết, Nhiều ràng buộc (Structured CoT + Prompt Chaining + Error Handling)
*Đặc điểm: Phân tách luồng xử lý thành các mắt xích (chain) rõ ràng, áp đặt định dạng đầu ra trung gian để chống đứt gãy logic và cài đặt cơ chế chặn lỗi biên (edge cases).*

```text
# VAI TRÒ (ROLE)
Bạn là Chuyên gia Xử lý Khiếu nại Cấp cao (Senior Escalation Specialist). Nhiệm vụ của bạn là thực thi quy trình xử lý khiếu nại theo chuỗi 3 bước nghiêm ngặt, đảm bảo tính chính xác tuyệt đối và không suy diễn.

# DỮ LIỆU ĐẦU VÀO
<customer_complaint>
{{INPUT_DATA}}
</customer_complaint>

# QUY TẮC NGHIỆP VỤ (CHÍNH SÁCH)
- Điều kiện 1: Sản phẩm bị lỗi kỹ thuật từ nhà sản xuất hoặc hư hỏng do vận chuyển.
- Điều kiện 2: Thời gian khiếu nại <= 7 ngày kể từ ngày nhận hàng.
- Điều kiện 3: Có cung cấp mã đơn hàng hợp lệ (bắt đầu bằng "ORD-").

# QUY TRÌNH XỬ LÝ CHUỖI (CHAIN-OF-THOUGHT & PROMPT CHAINING)
Hãy thực hiện tuần tự các bước dưới đây. Kết quả của bước trước là đầu vào bắt buộc cho bước sau.

## Bước 1: Trích xuất thực thể (Data Extraction)
Trích xuất các thông tin sau từ `<customer_complaint>`:
- Mã đơn hàng (`order_id`)
- Số ngày đã nhận hàng (`days_since_delivery`)
- Mô tả lỗi (`issue_description`)
*Ràng buộc chống đứt gãy:* Nếu thiếu bất kỳ trường thông tin nào, gán giá trị là `NULL`. Tuyệt đối không tự đoán.

## Bước 2: Đánh giá Logic & Xử lý lỗi biên (Logic Evaluation & Edge-case Handling)
Dựa trên dữ liệu ở Bước 1, đối chiếu với Quy tắc nghiệp vụ:
- **Trường hợp A (Lỗi biên - Thiếu dữ liệu):** Nếu có bất kỳ trường nào ở Bước 1 là `NULL` $\rightarrow$ DỪNG quy trình đánh giá hoàn tiền, chuyển trạng thái sang `NEED_MORE_INFO`.
- **Trường hợp B (Đủ dữ liệu):** Kiểm tra đồng thời cả 3 điều kiện. Chỉ kết luận `APPROVED` nếu thỏa mãn 100%, ngược lại kết luận `REJECTED` và nêu rõ điều kiện vi phạm.

## Bước 3: Khởi tạo phản hồi (Output Generation)
Dựa trên trạng thái ở Bước 2:
- Nếu `NEED_MORE_INFO`: Viết phản hồi lịch sự yêu cầu khách cung cấp chính xác các thông tin đang bị `NULL`.
- Nếu `APPROVED`: Thông báo chấp nhận hoàn tiền 100% và hướng dẫn gửi trả hàng.
- Nếu `REJECTED`: Từ chối khéo léo, trích dẫn rõ quy định vi phạm.

# ĐỊNH DẠNG ĐẦU RA BẮT BUỘC
Trình bày toàn bộ quá trình suy luận và kết quả theo cấu trúc JSON sau để hệ thống backend có thể đọc được:
{
  "step_1_extracted_data": { ... },
  "step_2_logic_trace": "Giải thích từng bước đối chiếu...",
  "status": "NEED_MORE_INFO | APPROVED | REJECTED",
  "step_3_final_response": "Nội dung gửi khách hàng..."
}
```

---

## 3. Bước 2: Lập bảng so sánh Trade-off

Để đánh giá tính hiệu quả trong môi trường vận hành thực tế, bảng dưới đây phân tích sự đánh đổi giữa hai giải pháp dựa trên các tiêu chí cốt lõi:

| Tiêu chí đánh giá | Giải pháp 1 (Đơn giản) | Giải pháp 2 (Chi tiết & Ràng buộc) | Phân tích Trade-off (Sự đánh đổi) |
| :--- | :--- | :--- | :--- |
| **Độ dễ hiểu (Readability)** | **Cao** | **Trung bình** | Giải pháp 1 dễ đọc, dễ viết nhanh cho người mới. Giải pháp 2 dài, chứa nhiều thẻ cấu trúc và logic điều kiện nên đòi hỏi người thiết kế phải có tư duy hệ thống. |
| **Tốn token (Token Cost)** | **Ít** (~60 tokens) | **Nhiều hơn** (~320 tokens) | Giải pháp 2 tiêu tốn lượng input token gấp ~5 lần Giải pháp 1, làm tăng chi phí API trên mỗi lượt gọi và tăng nhẹ độ trễ (latency). |
| **Chặn lỗi biên (Edge-case Handling)** | **Trung bình** | **Cực tốt** | Giải pháp 1 dễ bị "ảo giác" (tự bịa ra ngày nhận hàng hoặc tự ý đồng ý hoàn tiền khi khách than phiền gay gắt). Giải pháp 2 khóa chặt bằng biến `NULL` và rẽ nhánh `NEED_MORE_INFO`. |
| **Chống đứt gãy chuỗi (Chain Integrity)** | **Thấp** | **Rất cao** | Ở Giải pháp 1, mô hình có thể bỏ qua bước kiểm tra chính sách và nhảy thẳng tới việc viết email. Giải pháp 2 ép buộc đầu ra Bước 1 làm đầu vào Bước 2 thông qua cấu trúc JSON. |
| **Khả năng tích hợp hệ thống (Scalability)** | **Thấp** | **Cao** | Giải pháp 1 trả về văn bản tự do (unstructured text), khó bóc tách dữ liệu tự động. Giải pháp 2 trả về chuẩn JSON, sẵn sàng kết nối API. |

---

## 4. Bước 3: Lựa chọn phương án tối ưu và Chạy thử

### 4.1. Lựa chọn phương án và Lý do
* **Phương án được chọn:** **Giải pháp 2 (Chi tiết, có nhiều ràng buộc)**.
* **Lý do lựa chọn:** 
  1. Trong bài toán vận hành doanh nghiệp (đặc biệt liên quan đến tài chính/hoàn tiền), rủi ro từ việc xử lý sai quy trình hoặc hallucination gây thiệt hại lớn hơn rất nhiều so với chi phí của vài trăm token phát sinh thêm.
  2. Giải pháp 2 giải quyết triệt để bài toán **đứt gãy chuỗi (Prompt Chaining)** bằng cách cô lập từng bước suy luận (Trích xuất $\rightarrow$ Đánh giá $\rightarrow$ Phản hồi) và ép kiểu dữ liệu đầu ra (JSON), giúp kiểm soát hoàn toàn luồng tư duy của AI.

### 4.2. Kết quả chạy thử nghiệm (Simulation Test)

Để kiểm chứng khả năng chặn lỗi biên và chống đứt gãy của **Giải pháp 2**, ta sử dụng một dữ liệu đầu vào cố tình gài bẫy (khách hàng rất tức giận, mô tả lỗi rõ ràng, nhưng **thiếu mã đơn hàng** và **không nói rõ ngày nhận hàng**).

* **Dữ liệu đầu vào (`{{INPUT_DATA}}`):**
  > *"Làm ăn kiểu gì vậy hả? Tôi mới mua cái máy xay sinh tố của shop, cắm điện vào nó bốc khói khét lẹt rồi tắt ngúm luôn. Hàng lỗi thế này mà cũng giao cho khách được à? Hoàn tiền ngay lập tức cho tôi!"*

* **Kết quả đầu ra thực tế từ Mô hình (Output):**

```json
{
  "step_1_extracted_data": {
    "order_id": null,
    "days_since_delivery": null,
    "issue_description": "Máy xay sinh tố cắm điện bị bốc khói khét và tắt nguồn (lỗi kỹ thuật)."
  },
  "step_2_logic_trace": "Bước 1 trích xuất được lỗi kỹ thuật từ nhà sản xuất, nhưng thiếu 'order_id' (NULL) và 'days_since_delivery' (NULL). Đối chiếu Bước 2: Rơi vào Trường hợp A (Lỗi biên - Thiếu dữ liệu). Hệ thống dừng đánh giá điều kiện hoàn tiền và chuyển sang trạng thái NEED_MORE_INFO để tránh phê duyệt sai quy định.",
  "status": "NEED_MORE_INFO",
  "step_3_final_response": "Chào bạn, chúng tôi thành thật xin lỗi về trải nghiệm không tốt khi máy xay sinh tố gặp sự cố kỹ thuật. Để bộ phận hỗ trợ có thể kiểm tra và tiến hành thủ tục hoàn tiền theo chính sách, bạn vui lòng cung cấp giúp chúng tôi: (1) Mã đơn hàng (bắt đầu bằng ORD-) và (2) Thời gian bạn nhận được hàng nhé. Cảm ơn bạn!"
}
```

### 4.3. Đánh giá sau chạy thử
* **Tính chính xác:** Mô hình không bị cuốn theo cảm xúc tiêu cực của khách hàng để hứa hoàn tiền bừa bãi.
* **Xử lý đứt gãy & Lỗi biên:** Hệ thống nhận diện chính xác 2 trường dữ liệu bị khuyết (`null`), kích hoạt đúng nhánh `NEED_MORE_INFO` và tạo ra câu trả lời thu thập đúng thông tin còn thiếu. Quy trình chuỗi hoạt động trơn tru, không bị đứt gãy logic.