# Danh sách service & luồng chức năng dots.ocr

Tài liệu này tổng hợp các dịch vụ/dịch vụ phụ trợ đã có trong kho mã dots.ocr, mô tả chức năng chính, luồng dữ liệu chi tiết và các tham số cấu hình quan trọng để hỗ trợ mở rộng trong tương lai.

## 1. Dịch vụ suy luận OpenAI-compatible (vLLM)
**Chức năng:** triển khai mô hình dots.ocr trên nền vLLM và phơi bày API tương thích OpenAI Chat Completions tại `/:port/v1`.

**Luồng dữ liệu:**
1. Docker Compose dựng container `dots-ocr-server`, mount thư mục model nội bộ vào `/workspace/weights/DotsOCR` và đặt biến môi trường để Python ưu tiên module tuỳ biến.【F:docker/docker-compose.yml†L3-L40】
2. Điểm vào container sửa script `vllm` để import lớp mô hình tuỳ biến rồi chạy `vllm serve` với tham số tensor parallel, giới hạn bộ nhớ GPU và tên model được phục vụ (`dotsocr-model`).【F:docker/docker-compose.yml†L24-L40】
3. Ảnh gửi lên backend được client chuyển thành base64 và ghép với prompt trong khung Chat Completions; yêu cầu sử dụng prefix `<|img|><|imgpad|><|endofimg|>` để tránh xuống dòng tự động của vLLM.【F:dots_ocr/model/inference.py†L7-L45】

**Cấu hình:**
- Dockerfile kế thừa `vllm-openai:v0.9.1`, cài thêm FlashAttention 2.8.0 và Transformers 4.51.3 để tương thích với mô hình dots.ocr.【F:docker/Dockerfile†L1-L6】
- Compose mặc định lắng nghe `8000`, yêu cầu GPU ID `0` và gắn volume model cục bộ vào thư mục weights trong container.【F:docker/docker-compose.yml†L8-L19】

## 2. Thư viện lõi `DotsOCRParser` & CLI
**Chức năng:** xử lý end-to-end cho ảnh/PDF, bao gồm tiền xử lý, gọi mô hình (vLLM hoặc HuggingFace) và hậu xử lý đầu ra Markdown/JSON.

**Luồng dữ liệu:**
1. Khởi tạo parser cấu hình địa chỉ server, tên model, thông số sinh, DPI, thư mục output và giới hạn pixel; bật `use_hf` sẽ nạp model HuggingFace nội bộ và ép số luồng về 1.【F:dots_ocr/parser.py†L22-L60】【F:dots_ocr/parser.py†L62-L117】
2. Khi đọc file, `parse_file` chuẩn hoá thư mục kết quả, nhánh theo phần mở rộng để gọi `parse_pdf` hoặc `parse_image` rồi ghi JSONL tổng hợp.【F:dots_ocr/parser.py†L297-L322】
3. `parse_pdf` render từng trang PDF theo DPI cấu hình, lập danh sách tác vụ và xử lý song song bằng `ThreadPool`; kết quả được sắp xếp lại theo số trang.【F:dots_ocr/parser.py†L261-L295】
4. `_parse_single_image` nhận ảnh đã fetch, cân nhắc lại min/max pixels cho nhiệm vụ grounding, resize thông minh, tạo prompt, gọi backend và gom các đường dẫn đầu ra.【F:dots_ocr/parser.py†L133-L175】
5. Khi đầu ra là layout, pipeline cố gắng parse JSON, post-process bbox, vẽ ảnh minh hoạ, sinh Markdown (kèm bản no-hf). Nếu JSON lỗi, kết quả được làm sạch bởi `OutputCleaner` rồi đánh dấu `filtered` để UI biết sử dụng fallback text.【F:dots_ocr/parser.py†L178-L251】【F:dots_ocr/utils/layout_utils.py†L202-L228】【F:dots_ocr/utils/output_cleaner.py†L418-L435】

**Cấu hình:**
- CLI `main()` phơi bày đầy đủ tham số (đường dẫn đầu vào, chế độ prompt, bbox, giao thức, địa chỉ, tham số sampling, DPI, min/max pixels, bật/tắt fitz preprocess, chọn backend).【F:dots_ocr/parser.py†L326-L436】
- Hàm `_load_hf_model` đọc weights từ `./weights/DotsOCR`, sử dụng FlashAttention 2 và dtype bfloat16 khi chạy offline.【F:dots_ocr/parser.py†L62-L117】

**Khách hàng mẫu:**
- Script `demo_vllm.py` là client dòng lệnh tối giản để gửi ảnh mẫu qua API OpenAI của dịch vụ vLLM, phục vụ kiểm thử nhanh.【F:demo/demo_vllm.py†L1-L39】

## 3. Ứng dụng Gradio thời gian thực
**Chức năng:** giao diện một cửa cho ảnh/PDF, hỗ trợ xem kết quả từng trang, tải xuống zip và chuyển đổi cấu hình server ngay trong UI.【F:demo/demo_gradio.py†L32-L120】【F:demo/demo_gradio.py†L293-L377】

**Luồng dữ liệu:**
1. `DEFAULT_CONFIG` đặt IP, port, min/max pixels mặc định; một instance `DotsOCRParser` toàn cục tái sử dụng giữa các request.【F:demo/demo_gradio.py†L32-L52】
2. Hàm `process_image_inference` cập nhật cấu hình runtime, reset trạng thái session, quyết định đọc PDF hay ảnh và dùng `parse_pdf_with_high_level_api`/`parse_image_with_high_level_api` để gọi parser lõi.【F:demo/demo_gradio.py†L293-L400】
3. Với PDF, UI cache kết quả từng trang để người dùng chuyển trang mà không phải gọi lại backend; đồng thời đóng gói thư mục tạm thành zip để tải xuống.【F:demo/demo_gradio.py†L333-L377】
4. Với ảnh, nếu parser trả cờ `filtered` (JSON lỗi), giao diện chuyển sang hiển thị ảnh gốc + Markdown fallback; nếu thành công, ảnh layout, JSON và Markdown được render trong bảng thông tin.【F:demo/demo_gradio.py†L379-L399】【F:demo/demo_gradio.py†L386-L394】

**Cấu hình nâng cao:**
- UI cho phép bật `fitz_preprocess` để upscale DPI trước khi gửi đi, đồng thời cập nhật `min_pixels`/`max_pixels` của parser runtime.【F:demo/demo_gradio.py†L313-L325】【F:demo/demo_gradio.py†L296-L297】

## 4. Ứng dụng Gradio kèm annotate bbox
**Chức năng:** cho phép người dùng vẽ bbox trên ảnh và gửi kèm tới mô hình (prompt `prompt_grounding_ocr`).【F:demo/demo_gradio_annotion.py†L1-L158】

**Luồng dữ liệu:**
1. `process_annotation_data` chuẩn hoá dữ liệu từ widget `gradio_image_annotation`, chuyển ảnh về `PIL.Image` và lấy toạ độ bbox duy nhất.【F:demo/demo_gradio_annotion.py†L167-L197】
2. `parse_image_with_bbox` lưu ảnh tạm, gọi `DotsOCRParser.parse_image` với tham số `bbox`, rồi đọc lại JSON/Markdown/ảnh layout để trả về UI.【F:demo/demo_gradio_annotion.py†L97-L158】

**Cấu hình:** kế thừa `DEFAULT_CONFIG` và parser toàn cục giống bản realtime, nên có thể chia sẻ cấu hình server.【F:demo/demo_gradio_annotion.py†L32-L51】

## 5. Ứng dụng Gradio batch & pipeline hậu kỳ
**Chức năng:** xử lý nhiều tài liệu theo hàng đợi, hỗ trợ retry, ghi nhớ kết quả và cung cấp API scripting/exports cho hậu kỳ hàng loạt.【F:demo/demo_gradio_batch.py†L24-L176】【F:demo/demo_gradio_batch.py†L334-L420】

**Luồng dữ liệu:**
1. Cấu hình toàn cục giữ `DotsOCRParser` dùng chung, queue `TASK_QUEUE`, cache kết quả và tham số retry/backoff để điều phối nền.【F:demo/demo_gradio_batch.py†L24-L56】
2. Hàm `parse_image_with_high_level_api` (bản batch) lưu ảnh đầu vào, gọi parser và đọc lại mọi artifact (Markdown thường, Markdown no-hf, JSON, ảnh layout) để chuẩn bị cho UI/hậu kỳ.【F:demo/demo_gradio_batch.py†L196-L259】
3. `_set_parser_config` đồng bộ cấu hình server và giới hạn pixel cho toàn bộ worker trước khi đẩy job mới vào queue.【F:demo/demo_gradio_batch.py†L270-L287】
4. `purge_queue` cho phép huỷ job đang chờ; `ensure_export_ready`/`export_one_rid` gom file kết quả mỗi tài liệu thành zip phục vụ tải về hoặc script tự động.【F:demo/demo_gradio_batch.py†L289-L337】【F:demo/demo_gradio_batch.py†L304-L333】
5. `ScriptAPI` cung cấp bridge để chạy script Python tuỳ chỉnh trên tập kết quả (đọc trạng thái, lấy Markdown/JSON, tạo gói ExportBuilder).【F:demo/demo_gradio_batch.py†L338-L451】

**Cấu hình nâng cao:** người dùng có thể viết script riêng thông qua template `DEFAULT_SCRIPT_TEMPLATE`, lựa chọn dùng Markdown gốc hay bản chỉnh sửa/NOHF khi xuất dữ liệu.【F:demo/demo_gradio_batch.py†L56-L110】

## 6. Ứng dụng Streamlit
**Chức năng:** giao diện nhẹ dùng trực tiếp API vLLM (không qua `DotsOCRParser`), phù hợp demo nhanh hoặc tích hợp vào dashboard nội bộ.【F:demo/demo_streamlit.py†L32-L220】

**Luồng dữ liệu:**
1. Sidebar cho phép chọn prompt, IP/port server, min/max pixels; ảnh được đọc từ upload/URL/thư viện mẫu và cache để tránh tải lại nhiều lần.【F:demo/demo_streamlit.py†L32-L108】【F:demo/demo_streamlit.py†L62-L105】
2. Khi người dùng bấm “Start Inference”, ảnh sau tiền xử lý được gửi thẳng đến API OpenAI qua `inference_with_vllm`; kết quả và prompt được đóng gói thành dict để hiển thị.【F:demo/demo_streamlit.py†L203-L219】
3. `process_and_display_results` parse JSON, hậu xử lý bbox, tính kích thước input và hiển thị đồng thời ảnh layout + Markdown trong hai cột.【F:demo/demo_streamlit.py†L112-L166】

**Cấu hình:** sử dụng cùng giá trị mặc định (127.0.0.1:8000, min/max pixels) với các ứng dụng Gradio để thuận tiện đổi qua lại.【F:demo/demo_streamlit.py†L32-L75】

---
Các dịch vụ trên tận dụng chung lớp lõi `DotsOCRParser` và/hoặc API vLLM. Khi phát triển tính năng mới, bạn có thể tái sử dụng pipeline tiền/hậu xử lý của parser, hoặc kết nối trực tiếp tới dịch vụ vLLM bằng giao thức OpenAI để giữ tính thống nhất.
