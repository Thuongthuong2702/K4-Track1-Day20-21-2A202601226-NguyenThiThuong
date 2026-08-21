# AI Support Log

| # | Bước | AI dùng để làm gì | Bạn kiểm chứng kết quả thế nào |
|---|------|-------------------|-------------------------------|
| 1 | P2 Dataset v1 | Sinh 24 câu hỏi mô phỏng văn phong học viên (từ lóng "chấm tay", "vibe check", câu cộc lốc, giục deadline) bám theo 24 combination đã thiết kế | Đọc lại từng câu, gắn `metadata.slide` đúng slide nguồn, tự kiểm tra id/title khớp `manifest.json`; phát hiện 5 dòng nháp cũ còn sót trong `dataset.jsonl` cần dọn |
| 2 | P4 Judge prompt v1→v2 | Gợi ý sửa rubric groundedness (đổi "mọi khẳng định phải có nguồn trực tiếp" → "không mâu thuẫn với sources") và thêm ngoại lệ out-of-scope | Chạy lại `eval/judge.py`, so agreement trước/sau (42%→71%) với nhãn vàng đã chốt tay, không tin lời AI mà đo bằng số thật |
| 3 | Viết REPORT.md | Hỗ trợ diễn đạt lại số liệu thô (scorecard, confusion matrix) thành câu PM-friendly ở mục 6–7 | Đối chiếu từng số trong REPORT.md với file thô trong `deliverables/evidence/` trước khi chốt, sửa lại chỗ AI diễn giải sai/thổi phồng mức độ tin cậy |

## Phần AI gợi ý mà bạn bác bỏ

ở vòng calibrate judge, AI đề xuất hạ gate xuống 60% vì "70% cũng
coi là khá tốt" — đã bác bỏ vì gate phải chốt trước khi xem kết quả, hạ ngưỡng sau khi thấy
kết quả xấu là tự thương lượng, không phải calibration thật.

## Phần bạn hoàn toàn tự làm

Quyết định gate cuối cùng (≥80%) và verdict Hold/Ship; đối chiếu
8 case lệch và quyết định chốt nhãn vàng cho từng case; quyết định rubric và routing; chấm nhãn tay độc lập vòng 1.
