# Báo Cáo Tiến Độ Thực Hiện: Checkpoint 0 đến Checkpoint 5 (CP0 – CP5)

Tài liệu này tổng hợp toàn bộ các công việc, logic kỹ thuật, kết quả kiểm thử thực tế và cấu trúc mã nguồn đã hoàn thành từ **CP0 đến CP5** (bao gồm cả các bài tập Bonus) trong bài lab **AI Evaluation & Benchmarking Pipeline**.

---

## 1. Tổng quan trạng thái tiến độ

| Checkpoint | Hạng mục công việc | Trạng thái | Kết quả kiểm thử thực tế |
|---|---|---|---|
| **CP0** | Setup môi trường, `.env`, baseline test | **HOÀN THÀNH** | Môi trường Python 3.14 + `.venv`, 42 tests baseline |
| **CP1** | Data Models (`QAPair`, `EvalResult`, `overall_score`) | **HOÀN THÀNH** | 3/3 passed (`TestEvalResultOverallScore`) |
| **CP2** | RAGAS Metrics (Answer & Retrieval) + `LLMJudge` | **HOÀN THÀNH** | 21 passed, 20 failed, 1 skipped |
| **CP3** | `BenchmarkRunner` + `FailureAnalyzer` | **HOÀN THÀNH** | 41 passed, 1 skipped |
| **CP4** | Golden Dataset 20 QA, RAG run, Exercise 3.2 & 3.3 | **HOÀN THÀNH** | `validate_golden_dataset.py` PASS, artifacts sinh đầy đủ |
| **CP5** | Reflection, Failure Analysis, Bonus 3.4 & 3.5, Final Sync | **HOÀN THÀNH** | **42/42 passed** (100%), repo sạch, không lộ secret |

---

## 2. Chi tiết từng Checkpoint đã thực hiện

### CP0 — Setup & Baseline Môi trường
- Virtual environment `.venv` đã được tạo và kích hoạt.
- Toàn bộ dependencies trong `requirements.txt` đã cài đặt đầy đủ.
- File `.env` được cấu hình an toàn, hỗ trợ OpenAI, Google Gemini API và Local Endpoint (`http://localhost:20128/v1`).
- Đã thêm `.env` vào `.gitignore` để đảm bảo bảo mật.

---

### CP1 — Data Models (Task 1)
- Dataclass `QAPair`: Chứa đầy đủ các trường `question`, `expected_answer`, `context`, `metadata`, `retrieved_contexts`. Phân định rõ ràng, chống rò rỉ dữ liệu (data leakage).
- Dataclass `EvalResult`: Lưu trữ kết quả đánh giá cho từng câu trả lời thực tế, hỗ trợ 3 answer metrics và 2 retrieval metrics.
- Phương thức `overall_score()`: Tính trung bình cộng của `faithfulness`, `relevance`, `completeness`.

---

### CP2 — Metrics & LLM Judge (Tasks 2 & 3)
- `RAGASEvaluator`:
  - 3 Answer metrics: `evaluate_faithfulness`, `evaluate_relevance`, `evaluate_completeness`.
  - 2 Retrieval metrics: `evaluate_context_recall`, `evaluate_context_precision` ($AP@K$).
  - `run_full_eval()` kết nối 5 metrics và phân loại `failure_type` tự động.
- `LLMJudge`:
  - `score_response()`: Chấm điểm 1–5 theo rubric prompt và parse JSON an toàn.
  - `detect_bias()`: Phát hiện positional bias, leniency bias (>0.8), severity bias (<0.3).

---

### CP3 — Runner & Failure Analyzer (Tasks 4 & 5)
- `BenchmarkRunner`:
  - `run()`: Chạy batch QA pairs qua assistant và chuyển tiếp `retrieved_contexts`.
  - `generate_report()`: Tính pass rate, điểm trung bình các metrics và thống kê phân bổ lỗi.
  - `run_regression()`: Quality gate CI/CD, phát hiện sụt giảm điểm > 0.05.
  - `identify_failures()`: Lọc các kết quả thất bại theo threshold.
- `FailureAnalyzer`:
  - `categorize_failures()`: Phân loại lỗi.
  - `find_root_cause()`: Chẩn đoán nguyên nhân gốc rễ theo mô hình 5 Whys.
  - `generate_improvement_suggestions()`: Đề xuất ít nhất 3 hành động khắc phục cụ thể.
  - `generate_improvement_log()`: Xuất bảng Markdown chuẩn.

---

### CP4 — Golden Dataset & Real Benchmark
- `golden_dataset.json`: Hoàn thành chuẩn 20 QA pairs (5 Easy, 7 Medium, 5 Hard, 3 Adversarial) phủ 10/10 tài liệu trong `data/technology_store/`.
- Lệnh `python validate_golden_dataset.py` đạt **PASS**.
- Chạy `domain_assistant.py` sinh `artifacts/actual_answers.json` (20 câu trả lời thực tế).
- Chạy `evaluate_answers.py` sinh `artifacts/benchmark_results.json`.
- Hoàn thiện bảng kết quả 5 metrics và phân tích 3 cases thấp nhất trong `exercises.md` (Exercise 3.2).
- Thiết kế rubric domain OrbitTech và phương án kiểm soát bias trong `exercises.md` (Exercise 3.3).

---

### CP5 — Reflection, Bonus & Final Synchronization
- `reflection.md`:
  - Điền đầy đủ số liệu benchmark thực tế: pass rate 30.0%, Context Recall 0.925, Context Precision 0.947, Faithfulness 0.598, Relevance 0.535, Completeness 0.727.
  - Phân tích sâu 3 failure cases tiêu biểu (A01, A03, A02) theo phương pháp 5 Whys.
  - Bảng Failure Clustering và bảng Improvement Log từ `FailureAnalyzer`.
  - Chiến lược Regression Testing và Continuous Improvement Loop.
- **Bonus hoàn thành (+10 điểm)**:
  - **Exercise 3.4 (+5)**: Bảng so sánh chi tiết giữa RAGAS và DeepEval.
  - **Exercise 3.5 (+5)**: Triển khai hàm `rerank_by_overlap()` trong `template.py` và `solution/solution.py`. Chạy thử nghiệm trên 5 cases thực tế, chứng minh Context Precision tăng từ 0.938 lên 0.980 (+0.042) trong khi Context Recall giữ nguyên 1.000.
- **Đồng bộ mã nguồn**:
  - `template.py` và `solution/solution.py` đồng bộ 100% (`diff -u` không có khác biệt).
  - Lệnh `pytest tests/ -v` đạt **42/42 passed** (100% full suite kể cả bonus).
  - Không có file `.env` hay secret nào bị theo dõi bởi Git.

---

## 3. Checklist nghiệm thu tổng thể

- [x] Repository: Đúng chuẩn `K4-L3B-NguyenDinhTuanAnh-2A202602735-AIEvaluation`.
- [x] Unit Tests: **42 passed / 42 tests** (`pytest tests/ -v`).
- [x] Golden Dataset: **PASS** (`python validate_golden_dataset.py`, 20 QA, 10/10 docs).
- [x] Benchmark Run: Sinh thành công `actual_answers.json` và `benchmark_results.json`.
- [x] Exercises (`exercises.md`): Hoàn thành 100% từ Part 1 đến Part 3, bao gồm Exercise 3.4 & 3.5 bonus.
- [x] Reflection (`reflection.md`): Hoàn thành đầy đủ 7 mục phân tích sâu và bảng log.
- [x] Code Sync: `template.py` và `solution/solution.py` đồng bộ hoàn toàn.
- [x] Bảo mật: File `.env` được bảo vệ qua `.gitignore`.
