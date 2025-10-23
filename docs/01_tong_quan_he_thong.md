# Tài liệu tổng quan hệ thống dots.ocr

## 1. Bức tranh tổng thể
- **dots.ocr** là một bộ công cụ phân tích tài liệu đa ngôn ngữ xây dựng quanh lớp `DotsOCRParser`, cho phép xử lý ảnh và PDF để tạo ra thông tin bố cục, nội dung OCR và các bản xuất Markdown.【F:dots_ocr/parser.py†L17-L253】
- Hệ thống được thiết kế để gọi tới một máy chủ **OpenAI-compatible** (vLLM) hoặc tải trực tiếp mô hình HuggingFace, giúp linh hoạt trong triển khai on-premise hoặc dịch vụ.【F:dots_ocr/parser.py†L22-L131】【F:dots_ocr/model/inference.py†L7-L45】
- Các tiện ích xử lý hình ảnh, tài liệu, bố cục và chuyển đổi định dạng được gom trong `dots_ocr.utils`, đảm bảo pipeline tiền xử lý/hậu xử lý nhất quán.【F:dots_ocr/utils/image_utils.py†L1-L196】【F:dots_ocr/utils/layout_utils.py†L1-L228】【F:dots_ocr/utils/format_transformer.py†L1-L206】

## 2. Kiến trúc logic
```
Trình gọi (CLI/Demo/SDK)
        │
        ▼
  DotsOCRParser
        │  ├─ Chuẩn hóa input (fetch_image, smart_resize)
        │  ├─ Tạo prompt theo chế độ (dict_promptmode_to_prompt)
        │  └─ Điều phối backend (vLLM / HF)
        ▼
 Backend mô hình (OpenAI API over vLLM hoặc HF transformers)
        │
        ▼
 Hậu xử lý (post_process_output → layout_utils, format_transformer)
        │
        ▼
 Kết quả: JSON bố cục, ảnh gán bbox, Markdown, JSONL tổng hợp
```
- Bộ prompt định nghĩa các nhiệm vụ (layout full, layout only, OCR, grounding) và được parser lựa chọn thông qua khóa cấu hình.【F:dots_ocr/utils/prompts.py†L1-L34】【F:dots_ocr/parser.py†L133-L140】
- Pipeline hậu xử lý chuyển đổi phản hồi văn bản sang cấu trúc cells, vẽ bbox, và dựng Markdown để phục vụ benchmark hoặc xuất bản nội dung.【F:dots_ocr/utils/layout_utils.py†L31-L228】【F:dots_ocr/utils/format_transformer.py†L145-L206】

## 3. Thành phần cốt lõi
| Thành phần | Vai trò chính |
| --- | --- |
| `DotsOCRParser` | Điều phối toàn bộ chu trình: nhận input, tạo prompt, gọi backend, lưu kết quả (JSON/MD/ảnh/JSONL).【F:dots_ocr/parser.py†L143-L322】 |
| `model.inference` | Định nghĩa API gọi máy chủ vLLM tuân chuẩn OpenAI, quản lý thông số sinh token và khóa truy cập qua biến môi trường `API_KEY`.【F:dots_ocr/model/inference.py†L7-L45】 |
| `_load_hf_model` & `_inference_with_hf` | Cho phép chạy trực tiếp mô hình HuggingFace khi không có dịch vụ vLLM.【F:dots_ocr/parser.py†L62-L117】 |
| `utils.image_utils` | Đọc nhiều định dạng ảnh/URL/Base64, chuẩn hóa kích thước bằng `smart_resize` để khống chế số pixel gửi lên mô hình.【F:dots_ocr/utils/image_utils.py†L29-L140】 |
| `utils.doc_utils` | Chuyển PDF sang ảnh bằng PyMuPDF với DPI cấu hình để xử lý từng trang.【F:dots_ocr/utils/doc_utils.py†L20-L60】 |
| `utils.layout_utils` | Vẽ kết quả bố cục và chuẩn hóa lại toạ độ bbox từ không gian mô hình về ảnh gốc, đồng thời vệ sinh đầu ra bằng `OutputCleaner` khi JSON lỗi.【F:dots_ocr/utils/layout_utils.py†L31-L228】 |
| `utils.format_transformer` | Hợp nhất cells thành Markdown, xử lý LaTeX/tables và chuyển đổi ảnh thành base64 để nhúng.【F:dots_ocr/utils/format_transformer.py†L145-L206】 |
| Demo/CLI | Các script CLI, Gradio, Streamlit dùng `DotsOCRParser` làm API, minh hoạ cách nhúng vào ứng dụng khác.【F:README.md†L1131-L1174】【F:demo/demo_gradio.py†L1-L110】 |

## 4. Nguyên tắc thiết kế và đặc điểm
- **Prompt-centric multi-tasking:** Các chế độ làm việc chỉ khác nhau ở prompt, giúp mở rộng nhiệm vụ mà không phải sửa logic xử lý.【F:dots_ocr/utils/prompts.py†L1-L34】【F:dots_ocr/parser.py†L133-L253】
- **Tách biệt backend suy luận:** Lựa chọn `use_hf` cho phép chuyển đổi giữa vLLM và Transformers bằng cờ cấu hình, thuận tiện cho thử nghiệm và triển khai.【F:dots_ocr/parser.py†L53-L131】
- **Tối ưu hiệu năng tài liệu nhiều trang:** PDF được xử lý song song theo trang bằng `ThreadPool`, số luồng linh hoạt theo cấu hình và backend.【F:dots_ocr/parser.py†L261-L295】
- **Khả năng tự phục hồi đầu ra:** Khi mô hình trả về JSON lỗi, `OutputCleaner` sẽ dọn dẹp hoặc chuyển sang Markdown dạng văn bản để tránh mất dữ liệu.【F:dots_ocr/utils/layout_utils.py†L202-L228】【F:dots_ocr/utils/output_cleaner.py†L418-L435】

## 5. Đánh giá sẵn sàng sản xuất & kiến nghị
### Điểm mạnh
- **Module hoá rõ ràng:** Từng bước (tiền xử lý, suy luận, hậu xử lý) được đóng gói thành hàm riêng biệt giúp mở rộng và test độc lập.【F:dots_ocr/parser.py†L143-L253】【F:dots_ocr/utils/layout_utils.py†L115-L228】
- **Hỗ trợ triển khai đa môi trường:** Có sẵn Dockerfile/docker-compose và script đăng ký vLLM, giảm công sức vận hành.【F:docker/docker-compose.yml†L1-L40】【F:demo/launch_model_vllm.sh†L1-L17】
- **Kết quả đầu ra phong phú:** Parser lưu đồng thời JSON, Markdown, ảnh chú thích và JSONL tổng hợp, hỗ trợ nhiều kịch bản downstream.【F:dots_ocr/parser.py†L172-L322】

### Hạn chế & gợi ý
- **Xử lý lỗi còn đơn giản:** API vLLM mới chỉ bắt `RequestException` và trả `None`, chưa có cơ chế retry hay cảnh báo chuẩn hoá → nên bổ sung backoff và logging chuẩn hóa.【F:dots_ocr/model/inference.py†L19-L45】
- **Thiếu kiểm soát cấu hình tập trung:** Các tham số (đường dẫn model, DPI, số luồng) đang phân tán trong constructor hoặc script; nên gom vào file cấu hình hoặc biến môi trường để dễ quản trị.【F:dots_ocr/parser.py†L22-L58】【F:demo/launch_model_vllm.sh†L7-L15】
- **Chưa có kiểm thử/tự động hoá:** Repo không kèm pipeline kiểm thử; đề xuất bổ sung bộ kiểm tra hồi quy (ví dụ chạy parser trên bộ ảnh nhỏ) trước khi phát hành.
- **Bảo mật khóa truy cập:** `API_KEY` lấy trực tiếp từ biến môi trường nhưng mặc định "0" nếu thiếu, dễ gây nhầm lẫn; nên bắt buộc cấu hình rõ ràng hoặc thông báo cụ thể khi không tìm thấy khóa.【F:dots_ocr/model/inference.py†L19-L45】
