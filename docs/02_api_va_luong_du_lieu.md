# Tài liệu API & luồng dữ liệu dots.ocr

## 1. Khởi tạo `DotsOCRParser`
- Hàm khởi tạo nhận các tham số kết nối máy chủ (protocol, IP, port, tên model), cấu hình sinh nội dung (temperature, top_p, giới hạn token), tài nguyên xử lý (số luồng, DPI) và ràng buộc kích thước ảnh (min/max pixels).【F:dots_ocr/parser.py†L22-L52】
- Cờ `use_hf` cho phép nạp mô hình HuggingFace nội bộ thay vì gọi dịch vụ vLLM; khi bật, parser tự tải model và processor từ thư mục `./weights/DotsOCR`.【F:dots_ocr/parser.py†L53-L117】
- Bộ ràng buộc giá trị được kiểm tra ngay khi khởi tạo để đảm bảo số pixel nằm trong khoảng hợp lệ được định nghĩa trong `consts`.【F:dots_ocr/parser.py†L58-L60】【F:dots_ocr/utils/consts.py†L1-L6】

## 2. Luồng xử lý chính
### 2.1 `parse_file`
1. Chuẩn hoá đường dẫn đầu ra, tạo thư mục đích theo tên file nguồn.【F:dots_ocr/parser.py†L297-L308】
2. Phân nhánh theo phần mở rộng: gọi `parse_pdf` cho PDF hoặc `parse_image` cho ảnh; định dạng khác sẽ báo lỗi rõ ràng.【F:dots_ocr/parser.py†L310-L316】
3. Ghi log quá trình và tổng hợp kết quả từng trang vào file JSONL để tiện hậu kiểm/đánh giá batch.【F:dots_ocr/parser.py†L317-L322】

### 2.2 `parse_pdf`
1. Tải từng trang PDF thành ảnh với DPI cấu hình, lưu vào danh sách nhiệm vụ.【F:dots_ocr/parser.py†L262-L274】【F:dots_ocr/utils/doc_utils.py†L42-L60】
2. Chọn số luồng dựa trên backend (HF buộc 1 luồng), xử lý song song bằng `ThreadPool` và thanh tiến trình `tqdm`.【F:dots_ocr/parser.py†L279-L295】
3. Sắp xếp lại kết quả theo thứ tự trang và bổ sung đường dẫn gốc cho từng mục.【F:dots_ocr/parser.py†L292-L295】

### 2.3 `parse_image`
- Đọc ảnh bằng `fetch_image` (hỗ trợ đường dẫn, URL, base64) và chuyển sang `_parse_single_image` với tham số phù hợp.【F:dots_ocr/parser.py†L255-L259】【F:dots_ocr/utils/image_utils.py†L83-L140】

### 2.4 `_parse_single_image`
1. Quyết định lại giới hạn pixel khi làm nhiệm vụ grounding để đảm bảo hộp bbox chính xác.【F:dots_ocr/parser.py†L154-L159】
2. Tiền xử lý ảnh: tùy chọn tăng DPI qua `get_image_by_fitz_doc` trước khi resize chuẩn; kết quả `smart_resize` trả về kích thước đầu vào đã khống chế token.【F:dots_ocr/parser.py†L161-L167】【F:dots_ocr/utils/image_utils.py†L29-L140】
3. Xây dựng prompt theo chế độ tác vụ; với grounding, bbox được chuyển đổi về hệ toạ độ resize bằng `pre_process_bboxes`.【F:dots_ocr/parser.py†L133-L171】【F:dots_ocr/utils/layout_utils.py†L115-L145】
4. Thực thi backend: gọi `_inference_with_hf` hoặc `_inference_with_vllm` tương ứng cấu hình.【F:dots_ocr/parser.py†L168-L171】
5. Hậu xử lý:
   - Nếu tác vụ layout, phản hồi được parse JSON, quy đổi toạ độ về ảnh gốc (`post_process_cells`) và vẽ bbox lên ảnh; Markdown được sinh bằng `layoutjson2md` (kèm bản loại header/footer cho benchmark).【F:dots_ocr/parser.py†L178-L237】【F:dots_ocr/utils/layout_utils.py†L146-L228】【F:dots_ocr/utils/format_transformer.py†L145-L206】
   - Nếu mô hình trả về JSON lỗi, pipeline ghi thẳng phản hồi, đánh dấu `filtered=True` và vẫn dựng Markdown từ bản sạch đã vệ sinh bởi `OutputCleaner`.【F:dots_ocr/parser.py†L187-L237】【F:dots_ocr/utils/layout_utils.py†L202-L228】【F:dots_ocr/utils/output_cleaner.py†L418-L435】
   - Với các prompt chỉ cần text (OCR thuần), ảnh gốc được lưu kèm file Markdown chứa chuỗi trả về.【F:dots_ocr/parser.py†L238-L251】
6. Hàm trả về dict kết quả gồm số trang, kích thước đầu vào, đường dẫn file đầu ra và thông tin gốc để ghi log/đánh giá.【F:dots_ocr/parser.py†L172-L253】

## 3. Backend suy luận
### 3.1 vLLM/OpenAI API
- `inference_with_vllm` tạo client OpenAI tương thích, chuyển ảnh sang base64 và ghép prompt với tiền tố đặc thù `<|img|>` để đảm bảo định dạng mô hình; tham số nhiệt độ, top_p, max tokens có thể điều chỉnh linh hoạt.【F:dots_ocr/model/inference.py†L7-L40】
- Lỗi mạng được bắt qua `RequestException`, trả về `None` để caller xử lý; đề xuất bổ sung cơ chế retry phía trên để tránh mất kết quả.【F:dots_ocr/model/inference.py†L34-L45】

### 3.2 HuggingFace Transformers
- `_load_hf_model` nạp mô hình, processor và hàm `process_vision_info`, sử dụng Flash Attention 2 và dtype bfloat16 để tối ưu hiệu năng GPU.【F:dots_ocr/parser.py†L62-L76】
- `_inference_with_hf` dựng khung chat, chuyển dữ liệu sang GPU rồi gọi `generate` với giới hạn 24k token mới; đầu ra được giải mã và trả về chuỗi văn bản để pipeline hậu xử lý tương tự vLLM.【F:dots_ocr/parser.py†L78-L117】

## 4. Tạo prompt và quản lý chế độ
- Bản đồ `dict_promptmode_to_prompt` quy định nội dung chi tiết cho từng chế độ (layout đầy đủ, layout detection, OCR, grounding).【F:dots_ocr/utils/prompts.py†L1-L34】
- Hàm `get_prompt` chọn template tương ứng và xử lý bbox khi cần, đảm bảo backend luôn nhận toạ độ đã chuẩn hoá về kích thước input.【F:dots_ocr/parser.py†L133-L140】【F:dots_ocr/utils/layout_utils.py†L115-L145】

## 5. Hậu xử lý và định dạng đầu ra
- `post_process_output` cố gắng parse JSON từ phản hồi, chuyển đổi toạ độ, đồng thời fallback sang cơ chế vệ sinh chuỗi nếu gặp lỗi cú pháp.【F:dots_ocr/utils/layout_utils.py†L202-L228】
- `draw_layout_on_image` sử dụng PyMuPDF để render lớp chú thích bán trong suốt, thêm label theo thứ tự đọc để hỗ trợ kiểm tra kết quả.【F:dots_ocr/utils/layout_utils.py†L31-L113】
- `layoutjson2md` lặp qua từng cell, nhúng ảnh bằng base64, chuẩn hóa công thức LaTeX/tables để tạo Markdown phục vụ hiển thị hoặc benchmark.【F:dots_ocr/utils/format_transformer.py†L145-L206】
- File JSONL tổng hợp từ `parse_file` chứa toàn bộ metadata cho mỗi trang, thuận tiện cho đánh giá hàng loạt hoặc pipeline QA tự động.【F:dots_ocr/parser.py†L317-L322】

## 6. Tích hợp bên ngoài
- CLI `parser.py` cung cấp đầy đủ tham số (prompt, bbox, thông số kết nối, DPI, min/max pixels, lựa chọn backend) qua argparse để nhúng vào workflow tự động hoặc script tùy chỉnh.【F:dots_ocr/parser.py†L326-L435】
- Các ứng dụng Gradio/Streamlit sử dụng trực tiếp lớp parser, minh hoạ cách tích hợp pipeline vào giao diện người dùng thời gian thực.【F:demo/demo_gradio.py†L34-L110】【F:README.md†L1131-L1174】
