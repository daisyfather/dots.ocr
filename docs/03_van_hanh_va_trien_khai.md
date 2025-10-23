# Hướng dẫn vận hành & triển khai dots.ocr

## 1. Chuẩn bị môi trường
1. **Tạo môi trường Python 3.12 và cài gói**
   ```bash
   conda create -n dots_ocr python=3.12
   conda activate dots_ocr
   git clone https://github.com/rednote-hilab/dots.ocr.git
   cd dots.ocr
   pip install torch==2.7.0 torchvision==0.22.0 torchaudio==2.7.0 --index-url https://download.pytorch.org/whl/cu128
   pip install -e .
   ```
   Các bước trên được khuyến nghị trong tài liệu chính thức.【F:README.md†L993-L1005】
2. **Phụ thuộc bổ sung**: file `requirements.txt` liệt kê các thư viện phục vụ demo & hậu xử lý (Gradio, PyMuPDF, OpenAI SDK, Flash Attention…).【F:requirements.txt†L1-L11】
3. **Biến môi trường**: `inference_with_vllm` đọc khóa truy cập từ `API_KEY`; cần đặt giá trị hợp lệ khi máy chủ vLLM yêu cầu xác thực.【F:dots_ocr/model/inference.py†L19-L45】

## 2. Tải trọng mô hình
- Chạy script tải trọng: `python3 tools/download_model.py` (hoặc thêm `--type modelscope`). Thư mục đích mặc định `./weights/DotsOCR`; lưu ý sử dụng tên thư mục không chứa dấu chấm theo khuyến nghị trong README.【F:README.md†L1015-L1022】【F:tools/download_model.py†L1-L22】
- Khi vận hành offline, đặt biến `PYTHONPATH` trỏ tới thư mục chứa mô hình để vLLM có thể import mã tùy chỉnh.【F:demo/launch_model_vllm.sh†L7-L9】

## 3. Triển khai backend suy luận
### 3.1 vLLM (khuyến nghị)
- Khởi chạy máy chủ:
  ```bash
  vllm serve rednote-hilab/dots.ocr --trust-remote-code --async-scheduling --gpu-memory-utilization 0.95
  ```
  Lệnh này tạo API tương thích OpenAI phục vụ cho parser và demo.【F:README.md†L1029-L1037】
- Nếu cần đăng ký mô hình thủ công (vLLM < 0.11), sử dụng script `demo/launch_model_vllm.sh` để tải trọng, vá entrypoint và khởi chạy với cấu hình GPU phù hợp.【F:demo/launch_model_vllm.sh†L1-L15】

### 3.2 HuggingFace Transformers
- Có thể suy luận trực tiếp qua `python3 demo/demo_hf.py`; parser cũng hỗ trợ cờ `--use_hf true` để gọi hàm `_inference_with_hf` thay vì vLLM.【F:README.md†L1039-L1056】【F:dots_ocr/parser.py†L53-L117】
- Chế độ này phù hợp cho thử nghiệm hoặc môi trường không có dịch vụ OpenAI-compatible, đổi lại tốc độ chậm hơn (README lưu ý rõ).【F:README.md†L1151-L1153】

### 3.3 Docker & Docker Compose
- Dockerfile kế thừa `vllm/vllm-openai:v0.9.1`, cài thêm `flash_attn` và `transformers` để hỗ trợ đăng ký mô hình ngoài cây chính.【F:docker/Dockerfile†L1-L6】
- `docker/docker-compose.yml` cấu hình container tên `dots-ocr-container`, ánh xạ cổng 8000, mount thư mục mô hình và vá script `vllm` trước khi chạy `vllm serve`. Cần GPU (device 0) và biến `PYTHONPATH` tới thư mục weights.【F:docker/docker-compose.yml†L3-L40】

## 4. Thực thi phân tích tài liệu
### 4.1 CLI `parser.py`
- Ví dụ phổ biến (yêu cầu backend vLLM đang chạy):
  ```bash
  # Layout + OCR đầy đủ
  python3 dots_ocr/parser.py demo/demo_image1.jpg
  # Phân tích PDF nhiều trang (tăng số luồng nếu cần)
  python3 dots_ocr/parser.py demo/demo_pdf1.pdf --num_thread 64
  # Layout detection only
  python3 dots_ocr/parser.py demo/demo_image1.jpg --prompt prompt_layout_only_en
  # Chỉ lấy text (bỏ header/footer)
  python3 dots_ocr/parser.py demo/demo_image1.jpg --prompt prompt_ocr
  # Grounding theo bbox
  python3 dots_ocr/parser.py demo/demo_image1.jpg --prompt prompt_grounding_ocr --bbox 163 241 1536 705
  ```
  Các lệnh trên được tài liệu chính thức minh họa; thêm `--use_hf true` nếu muốn chạy bằng Transformers.【F:README.md†L1131-L1153】
- Tham số đáng chú ý:
  - `--output`: thư mục lưu kết quả (mặc định `./output`).
  - `--min_pixels/--max_pixels`: khống chế kích thước ảnh gửi mô hình, nên dùng khi tài liệu rất lớn.【F:dots_ocr/parser.py†L297-L322】【F:dots_ocr/parser.py†L326-L400】
  - `--no_fitz_preprocess`: bỏ bước tăng DPI bằng PyMuPDF nếu ảnh đầu vào đã đủ rõ.【F:dots_ocr/parser.py†L389-L395】

### 4.2 Ứng dụng demo
- Chạy Gradio demo tổng quát hoặc bản ground-truth annotation:
  ```bash
  python demo/demo_gradio.py
  python demo/demo_gradio_annotion.py
  ```
  Các demo này xây dựng phiên làm việc quanh `DotsOCRParser` và cung cấp giao diện tải file, xem kết quả trực quan.【F:README.md†L1165-L1174】【F:demo/demo_gradio.py†L34-L110】
- Ngoài ra có notebook Colab, script batch, Streamlit… trong thư mục `demo/` phục vụ các kịch bản thử nghiệm khác.

## 5. Quản trị & giám sát
- **Log và kết quả**: `parse_file` tạo thư mục cùng tên file nguồn, chứa ảnh chú thích (`*.jpg`), JSON bố cục (`*.json`), Markdown (`*.md`, `*_nohf.md`) và file tổng hợp `*.jsonl`. Điều này thuận tiện cho việc kiểm thử, so sánh benchmark và truyền dữ liệu sang pipeline downstream.【F:dots_ocr/parser.py†L172-L322】
- **Vệ sinh đầu ra**: Khi gặp phản hồi JSON lỗi, parser tự động rơi về chế độ `filtered`, lưu cả bản gốc và bản sạch sau khi `OutputCleaner` xử lý để tránh mất thông tin. Nên kiểm tra cờ này trong các workflow tự động để quyết định tái chạy hay chấp nhận kết quả.【F:dots_ocr/parser.py†L187-L237】【F:dots_ocr/utils/output_cleaner.py†L418-L435】
- **Bảo trì mô hình**: Khi nâng cấp vLLM ≥ 0.11, không cần vá entrypoint nữa vì dots.ocr đã được tích hợp chính thức; chỉ cần kéo image mới và mount weights tương ứng.【F:README.md†L1025-L1037】

## 6. Checklist triển khai sản xuất
1. Tải và xác thực mô hình (`tools/download_model.py`, kiểm tra checksum nếu có).
2. Cấu hình dịch vụ vLLM (GPU, `API_KEY`, `PYTHONPATH`, giới hạn bộ nhớ) và kiểm thử bằng `demo/demo_vllm.py` trước khi mở API cho người dùng.【F:README.md†L1029-L1037】【F:demo/demo_vllm.py†L1-L60】
3. Thiết lập cơ chế giám sát log & lỗi (xử lý `None` trả về từ `inference_with_vllm`).【F:dots_ocr/model/inference.py†L34-L45】
4. Tích hợp CI/CD để chạy regression nhỏ (ví dụ dùng `demo/demo_image1.jpg`) nhằm phát hiện thay đổi bất thường của mô hình.
5. Với môi trường nhiều người dùng, cân nhắc triển khai docker-compose hoặc Kubernetes dựa trên cấu hình sẵn có để dễ scale-out.【F:docker/docker-compose.yml†L3-L40】
