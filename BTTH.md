# BÁO CÁO THIẾT KẾ GIẢI PHÁP PROMPTING NÂNG CAO (BÀI TẬP TỔNG HỢP)

## 1. Bối cảnh bài toán thực tế
**Bài toán:** Xây dựng hệ thống AI Assistant tự động tiếp nhận và xử lý yêu cầu **"Bảo hành/Đổi trả sản phẩm"** từ khách hàng cho một hệ thống bán lẻ. 
**Vấn đề:** Khách hàng thường nhắn tin lộn xộn, hay quên cung cấp "Mã đơn hàng" hoặc "Tình trạng lỗi". 

## 2. Thiết kế giải pháp (Architecture)
*   **Bước 1: Data Extractor & Router (Trích xuất & Điều hướng)**
    *   *Nhiệm vụ:* Đọc tin nhắn, suy luận từng bước (CoT) để tìm kiếm 2 thông tin bắt buộc: `Mã đơn hàng` và `Tình trạng lỗi`.
    *   *Xử lý đứt gãy:* Nếu thiếu 1 trong 2, đánh dấu trạng thái lỗi (`MISSING_DATA`) để chặn việc gọi API backend.
    *   *Đầu ra:* Chuẩn hóa dưới dạng JSON.
*   **Bước 2: Action & Response Generator (Thực thi & Phản hồi)**
    *   *Nhiệm vụ:* Nhận đầu vào là mã JSON từ Bước 1. Dựa vào trạng thái (Status) để quyết định viết email phản hồi xin thêm thông tin (duy trì hội thoại) hay thông báo thành công.

## 3. Hệ thống Prompt thực thi (System Prompts)

### 🔴 PROMPT BƯỚC 1: Data Extractor (Áp dụng Chain-of-Thought & Bắt lỗi)
```text
Bạn là một hệ thống AI phân tích dữ liệu khách hàng.
Nhiệm vụ của bạn là đọc tin nhắn đầu vào, suy luận từng bước và trích xuất dữ liệu.

[INPUT]
Tin nhắn: "{user_message}"

[QUÁ TRÌNH TƯ DUY - CHAIN OF THOUGHT]
Hãy phân tích ngầm theo các bước sau trước khi xuất kết quả:
- Bước 1: Khách hàng đang gặp vấn đề gì?
- Bước 2: Tìm "Mã đơn hàng". Khách đã cung cấp chưa?
- Bước 3: Tìm "Tình trạng lỗi". Khách đã mô tả sản phẩm bị sao chưa?
- Bước 4: Đánh giá toàn vẹn dữ liệu. Nếu có đủ Mã đơn và Tình trạng, status là "VALID". Nếu thiếu, status là "MISSING_DATA".

[RÀNG BUỘC ĐẦU RA]
Chỉ xuất ra KẾT QUẢ DUY NHẤT dưới định dạng JSON nguyên bản:
{
  "order_id": "[Mã đơn hàng hoặc null]",
  "issue_description": "[Mô tả lỗi hoặc null]",
  "status": "[VALID hoặc MISSING_DATA]",
  "missing_fields": "[Danh sách các trường bị thiếu]"
}
```

### 🔵 PROMPT BƯỚC 2: Action Generator (Áp dụng Prompt Chaining)
```text
Bạn là Chuyên viên CSKH cao cấp. Hãy đọc dữ liệu hệ thống cung cấp dưới dạng JSON và sinh ra phản hồi giao tiếp với khách hàng.

[INPUT TỪ HỆ THỐNG]
{json_output_from_step_1}

[LUỒNG XỬ LÝ ĐỨT GÃY NGỮ CẢNH]
1. Nếu "status" == "MISSING_DATA": 
   - Hãy viết một email phản hồi thân thiện, xin lỗi về sự cố khách gặp phải.
   - Yêu cầu khách hàng cung cấp bổ sung chính xác các thông tin đang nằm trong trường "missing_fields".
2. Nếu "status" == "VALID":
   - Viết email thông báo đã tiếp nhận thành công.
   - Xác nhận lại [order_id] và [issue_description].
   - Thông báo bộ phận kỹ thuật sẽ liên hệ trong 24h.

[RÀNG BUỘC ĐẦU RA]
Viết bằng tiếng Việt, giọng điệu chuyên nghiệp, thấu cảm.
```

## 4. Kết quả chạy thử (Test Cases)

### Test Case 1: Dữ liệu bị đứt gãy (Khách quên nhập Mã đơn hàng)
*   **User Input:** *"Chào shop, hôm qua tôi nhận được cái tai nghe nhưng cắm sạc không vào điện. Đổi cho tôi cái mới ngay!"*
*   **Output Bước 1 (JSON):** `{"order_id": null, "issue_description": "Tai nghe cắm sạc không vào điện", "status": "MISSING_DATA", "missing_fields": "Mã đơn hàng"}`
*   **Output Bước 2 (Phản hồi cuối cùng của AI):**
    > "Chào bạn, Cửa hàng rất xin lỗi vì sự bất tiện bạn gặp phải khi tai nghe cắm sạc không vào điện. Để chúng tôi có thể tra cứu hệ thống và hỗ trợ đổi trả sản phẩm cho bạn nhanh nhất, bạn vui lòng cung cấp bổ sung giúp cửa hàng **Mã đơn hàng** của bạn nhé. Ngay khi nhận được thông tin, bộ phận CSKH sẽ tiến hành xử lý ngay ạ. Cảm ơn bạn!"

### Test Case 2: Luồng dữ liệu hoàn hảo (Happy Path)
*   **User Input:** *"Shop bảo hành giúp mình đơn #998877 nhé, màn hình điện thoại bị sọc ngang rồi."*
*   **Output Bước 1 (JSON):** `{"order_id": "#998877", "issue_description": "Màn hình điện thoại bị sọc ngang", "status": "VALID", "missing_fields": ""}`
*   **Output Bước 2 (Phản hồi cuối cùng của AI):**
    > "Chào bạn, Cửa hàng đã tiếp nhận thành công yêu cầu bảo hành cho đơn hàng **#998877** với tình trạng lỗi được ghi nhận là: **Màn hình điện thoại bị sọc ngang**. Thông tin của bạn đã được chuyển đến bộ phận kỹ thuật. Chúng tôi sẽ liên hệ lại với bạn trong vòng 24h tới để hướng dẫn chi tiết các bước gửi trả hàng. Xin lỗi bạn vì trải nghiệm không mong muốn này!"
