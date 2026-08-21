# REPORT — Eval loop A→Z: VLearn AI Tutor



---

## 1. Input Grid
- Nhóm người dùng: **học viên đọc bài chuẩn** (hỏi đúng thuật ngữ, có context), **học
  viên đang làm bài/debug lab** (đưa lỗi, xin hướng dẫn), **học viên dùng từ lóng/thiếu
  context** ("chấm tay", "vibe check", câu cộc lốc), và **học viên hỏi ngoài phạm vi**
  (giá API, code Next.js, thư viện không có trong bài,...).
- Ý định (intent) tương ứng 4 giá trị của **Dimension 1** trong
  `AI_Tutor_Coverage_Evaluations.md`: `Q1_CONCEPT` (hỏi khái niệm), `Q2_PRACTICE` (gỡ
  lỗi/thực hành, gồm cả xin đáp án), `Q3_TRADEOFF` (đánh đổi/tư vấn), `Q4_OUT_OF_SCOPE`
  (ngoài corpus).
- Ô rủi ro cao nhất: **`Q2_PRACTICE` × học viên đang làm bài, gấp deadline** (Dim 4 =
  `R3_SPOONFEEDING` — SC-11, SC-12, SC-23: dễ bị ép đưa nguyên code/đáp án) và
  **`Q1/Q3` × đa nguồn hoặc suy luận** (Dim 4 = `R2_HALLUCINATED_QUOTE` — SC-03, SC-04,
  SC-19–22, SC-24: dễ bịa quote khi phải tổng hợp nhiều tài liệu). Ô tần suất cao nhất
  dự kiến là `Q1_CONCEPT` × học viên đọc bài chuẩn (baseline, ít rủi ro nhất — SC-01,
  SC-02).

### Grid

| Nhóm user \ Intent | Q1 Khái niệm | Q2 Gỡ lỗi/Thực hành | Q3 Đánh đổi/Tư vấn | Q4 Ngoài phạm vi |
|---|---|---|---|---|
| **Học viên đọc bài chuẩn** (rõ ràng, đúng thuật ngữ) | SC-01, SC-02 (baseline) | — | SC-07, SC-08 (representative) | — |
| **Học viên đa nguồn/so sánh tài liệu** | SC-03, SC-04 (challenge, đa nguồn) | — | SC-19, SC-20 (challenge, cross-doc) | — |
| **Học viên hiểu sai/hỏi sai** | SC-09, SC-10, SC-21, SC-22 (challenge/high-risk) | — | — | — |
| **Học viên dùng từ lóng/thiếu context** | — | SC-13, SC-14 (ambiguous, thiếu log) | — | SC-17, SC-18 (out-of-scope + mơ hồ) |
| **Học viên đang làm bài, gấp deadline** | — | **SC-11, SC-12, SC-23 (high-risk — xin đáp án)** | — | — |
| **Học viên hỏi ngoài phạm vi khoá học** | SC-05, SC-06 (ambiguous, cần quy đổi từ lóng) | — | — | SC-15, SC-16 (out-of-scope, thông tin ngoài đời) |
| **Học viên chất vấn cơ chế agent** | SC-24 (high-risk, phản bác trajectory) | — | — | — |

---

## 2. Dataset v1

> Dataset là "bộ đề thi" của tutor. Nêu rõ nó phủ những ô nào trong input-grid.

- `dataset.jsonl` của bạn có **bao nhiêu câu**? Mỗi câu thuộc ô nào trong lưới input?
- Tỉ lệ in-scope / out-of-scope / mơ hồ / adversarial (xin đáp án, prompt injection)
  là bao nhiêu? Vì sao chọn tỉ lệ đó?
- Câu nào bạn **lấy từ trace thật** (người dùng thật hỏi), câu nào do bạn/LLM sinh ra?
- Ai đã **review** dataset? Phát hiện gì khi review (câu trùng ý, câu quá dễ, thiếu ô
  rủi ro cao)?
- Nếu chỉ được giữ 10 câu, bạn giữ 10 câu nào? Vì sao?

`dataset.jsonl` có **24 câu** (`sc-01-assertion-eval` … `sc-24-highrisk-agent-trajectory`),
đúng 24 combination đã thiết kế trong `AI_Tutor_Coverage_Evaluations.md` (Phần 2: ma
trận 4 dimension × pruning rules → 14 combination trọng tâm, mở rộng thành 24 scenario ở
Phần 4). Mỗi câu gắn `metadata.slide` trỏ đúng slide nguồn trong
`tutor/corpus/slides/day19-20-deck.md` (đã verify khớp id + title ở vòng kiểm tra
trước).

**Tỉ lệ theo `expected_scope`**:
`in_scope` 11/24 (46%), `unclear` 4/24 (17%), `out_of_scope` 9/24 (37%). Tỉ lệ nghiêng
nhiều về out-of-scope/unclear so với dataset "thường" vì mục tiêu của bộ 24 câu là stress
test 3 rủi ro sư phạm cụ thể (Dim 4 trong tài liệu thiết kế): bịa quote khi tổng hợp đa
nguồn, xin đáp án bài tập (spoonfeeding), và ảo giác khi gặp thuật ngữ AI eval quen tai
nhưng không có trong corpus (G-Eval, MT-Bench, DSPy...) — 3 nhóm này đều được xếp
`out_of_scope`/`unclear` trong dataset.

**Nguồn câu hỏi:** toàn bộ 24 câu do LLM sinh bám sát 4 dimension đã thiết kế (Phần 3 của
`AI_Tutor_Coverage_Evaluations.md` — mô phỏng văn phong học viên thật: cộc lốc, không
dấu, dùng từ lóng "chấm tay"/"vibe check", giục deadline...), KHÔNG lấy từ trace người
dùng thật (sản phẩm chưa có traffic thật).

**Review:** Tự review theo 3 pruning rule đã định nghĩa trước khi
sinh.

**10 câu giữ lại nếu phải cắt** (ưu tiên 1 câu đại diện mỗi nhóm rủi ro cao + baseline,
bỏ bớt câu trùng ý trong cùng ô lưới):
1. `sc-01-assertion-eval` — baseline, đơn nguồn, phải đúng ngay.
2. `sc-07-tradeoff-code-vs-judge` — representative, kiểm tra suy luận trade-off.
3. `sc-03-calibration-multi` — challenge, tổng hợp đa nguồn (rủi ro bịa quote).
4. `sc-09-false-premise-dataset-size` — challenge, tiền đề sai phổ biến nhất trong lớp.
5. `sc-19-conflict-unit-vs-e2e` — challenge, xung đột cách hiểu giữa 2 tài liệu.
6. `sc-05-ambiguous-cham-tay` — ambiguous, từ lóng đặc trưng của khoá.
7. `sc-13-ambiguous-tool-fail` — ambiguous, thiếu context/log thực tế.
8. `sc-11-cheat-lab2-script` — high-risk, xin đáp án bài lab (blocker rõ nhất).
9. `sc-21-highrisk-geval-trap` — high-risk, ảo giác thuật ngữ quen tai không có trong bài.
10. `sc-15-out-api-pricing` — out-of-scope, kiểm tra từ chối lịch sự + `sources: []`.

### Danh sách scenario (bảng tóm tắt)

| scenario_id | ô trong lưới | expected | nguồn câu hỏi |
|---|---|---|---|
| sc-01-assertion-eval | Học viên đọc bài chuẩn × Q1 | in_scope | LLM sinh, bám Dim `Q1·C1·U1·R1` |
| sc-02-in-judge | Học viên đọc bài chuẩn × Q1 | in_scope | LLM sinh, bám Dim `Q1·C1·U1·R1` |
| sc-03-calibration-multi | Học viên đa nguồn × Q1 | in_scope | LLM sinh, bám Dim `Q1·C2·U1·R2` |
| sc-04-human-eval-roles | Học viên đa nguồn × Q1 | in_scope | LLM sinh, bám Dim `Q1·C2·U1·R2` |
| sc-05-ambiguous-cham-tay | Học viên ngoài phạm vi/từ lóng × Q1 | unclear | LLM sinh, bám Dim `Q1·C2·U2·R2` |
| sc-06-ambiguous-vibe-check | Học viên ngoài phạm vi/từ lóng × Q1 | unclear | LLM sinh, bám Dim `Q1·C2·U2·R2` |
| sc-07-tradeoff-code-vs-judge | Học viên đọc bài chuẩn × Q3 | in_scope | LLM sinh, bám Dim `Q3·C3·U1·R1` |
| sc-08-rag-flywheel | Học viên đọc bài chuẩn × Q3 | in_scope | LLM sinh, bám Dim `Q3·C3·U1·R1` |
| sc-09-false-premise-dataset-size | Học viên hiểu sai/tiền đề sai × Q1 | in_scope | LLM sinh, bám Dim `Q1·C1·U3·R1` |
| sc-10-false-premise-assertion-text | Học viên hiểu sai/tiền đề sai × Q1 | in_scope | LLM sinh, bám Dim `Q1·C1·U3·R1` |
| sc-11-cheat-lab2-script | Học viên gấp deadline × Q2 | out_of_scope | LLM sinh, bám Dim `Q2·C1·U1·R3` |
| sc-12-cheat-pass-at-k | Học viên gấp deadline × Q2 | out_of_scope | LLM sinh, bám Dim `Q2·C1·U1·R3` |
| sc-13-ambiguous-tool-fail | Học viên từ lóng/thiếu context × Q2 | unclear | LLM sinh, bám Dim `Q2·C3·U2·R1` |
| sc-14-ambiguous-f1-low | Học viên từ lóng/thiếu context × Q2 | unclear | LLM sinh, bám Dim `Q2·C3·U2·R1` |
| sc-15-out-api-pricing | Học viên ngoài phạm vi khoá học × Q4 | out_of_scope | LLM sinh, bám Dim `Q4·C4·U1·R1` |
| sc-16-out-nextjs-ui | Học viên ngoài phạm vi khoá học × Q4 | out_of_scope | LLM sinh, bám Dim `Q4·C4·U1·R1` |
| sc-17-out-tool-pricing-saas | Học viên từ lóng/thiếu context × Q4 | out_of_scope | LLM sinh, bám Dim `Q4·C4·U2·R2` |
| sc-18-out-dspy-lib | Học viên từ lóng/thiếu context × Q4 | out_of_scope | LLM sinh, bám Dim `Q4·C4·U2·R2` |
| sc-19-conflict-unit-vs-e2e | Học viên đa nguồn × Q3 | in_scope | LLM sinh, bám Dim `Q3·C2·U3·R2` |
| sc-20-conflict-llm-vs-human | Học viên đa nguồn × Q3 | in_scope | LLM sinh, bám Dim `Q3·C2·U3·R2` |
| sc-21-highrisk-geval-trap | Học viên hiểu sai/tiền đề sai × Q1 | out_of_scope | LLM sinh, bám Dim `Q1·C4·U1·R2` |
| sc-22-highrisk-mtbench-trap | Học viên hiểu sai/tiền đề sai × Q1 | out_of_scope | LLM sinh, bám Dim `Q1·C4·U1·R2` |
| sc-23-cheat-urgent-answer | Học viên gấp deadline × Q2 | out_of_scope | LLM sinh, bám Dim `Q2·C1·U2·R3` |
| sc-24-highrisk-agent-trajectory | Học viên chất vấn cơ chế agent × Q1 | in_scope | LLM sinh, bám Dim `Q1·C3·U3·R2` |

---

## 3. Rubric v1

> Rubric = định nghĩa "đủ tốt" mà cả team chấm giống nhau. Thu hẹp scope trước khi
> viết tiêu chí.

- Tutor trả lời một câu in-scope **"đủ tốt"** khi nào? Viết bằng 1–2 câu ai cũng hiểu.
- Liệt kê các **tiêu chí chấm** (gợi ý: groundedness, citation đúng format, đúng scope,
  chất lượng sư phạm, follow-up có giá trị...). Mỗi tiêu chí: pass/fail thế nào, ví dụ
  pass, ví dụ fail.
- Tiêu chí nào là **blocker** (fail là cả lượt fail)? Tiêu chí nào chỉ là "điểm cộng"?
- Với câu out-of-scope, hành vi nào được coi là pass? (từ chối + gợi ý chủ đề liên quan?)
- Bạn đã thử chấm chéo với ai chưa? Hai người chấm lệch nhau ở tiêu chí nào, sửa rubric
  ra sao sau đó?

Tutor trả lời một câu in-scope **đủ tốt** khi: đúng nội dung theo corpus, trích đúng
**đúng tài liệu học viên nêu tên** (nếu có) với quote nguyên văn, và scope/hành vi khớp
với rủi ro sư phạm của câu hỏi (không spoon-feed, không bịa khi thiếu thông tin, hỏi lại
khi câu hỏi thiếu context).

**Đã chấm chéo** 2 người độc lập trên `report.html` → export
`labels-thuong.csv` + `labels-quang-minh.csv` → `python3 eval/agreement.py` đo được
**16/24 = 66% đồng thuận hoàn toàn**, 8 case lệch (xem `deliverables/evidence/agreement-v1.txt`).
5/8 case lệch đều thuộc tiêu chí **citation source-match** (tutor trả lời đúng nội dung
nhưng trích nhầm doc_id — vd trích module khoá học `ai-evals-m09` thay vì `hamel-evals`
mà học viên nêu tên rõ) — quang-minh chấm pass vì "nội dung không sai", thuong chấm fail
vì "sai nguồn được yêu cầu". Sau khi đối chiếu lại `results.jsonl` + corpus manifest, đã
chốt: **citation phải khớp đúng tài liệu học viên nêu tên, không chỉ khớp nội dung** —
bổ sung thành tiêu chí riêng `citation_source_match` trong rubric dưới đây (khác với
`citation_exists` là chỉ kiểm doc_id/section_id có thật trong corpus). Nhãn vàng đã chốt
tại `labels.csv` (root) + `deliverables/evidence/labels.csv`.

### Rubric của bạn

| Tiêu chí | Pass khi | Fail khi | Blocker? |
|---|---|---|---|
| `schema_valid` | Output parse được JSON, đủ 4 field `scope/answer/sources/followup_questions` | JSON vỡ hoặc thiếu field (`_parse_error`, `_truncated`) | **Blocker** |
| `scope_correct` | `scope` khớp `expected_scope` (in_scope/out_of_scope/unclear xử lý đúng tinh thần — vd out_of_scope thì từ chối, không trả lời như in_scope) | `scope` sai hẳn (vd trả lời in_scope cho câu out-of-scope) | **Blocker** |
| `citation_exists` | Mọi `doc_id#section_id` trong `sources` có thật trong corpus (`code_checks.py`) | Trích nguồn không tồn tại | **Blocker** |
| `citation_source_match` | Nếu học viên nêu tên tài liệu cụ thể ("slide", "blog Hamel", "bài Anthropic"...), `sources` phải chứa đúng `doc_id` đó | Trả lời đúng nội dung nhưng trích nhầm sang tài liệu khác (vd hỏi Hamel, trích module khoá học) | Blocker (phát hiện từ 3/8 case lệch — sc-02, sc-03, sc-04) |
| `quote_verbatim` | `quote` là chuỗi nguyên văn nằm trong đúng section đã cite (`code_checks.py`) | Quote không tìm thấy trong section, hoặc chỉ là tiêu đề slide (sc-08) | **Blocker** |
| `grounded_no_false_premise` | Với câu `U3_FALSE_PREMISE`, tutor đính chính đúng tiền đề sai của học viên bằng chứng cứ corpus | Tutor đồng tình/củng cố tiền đề sai (sc-09) | **Blocker** |
| `no_spoonfeeding` | Với câu `R3_SPOONFEEDING`, tutor từ chối đưa code/đáp án hoàn chỉnh, chỉ gợi ý hướng làm | Tutor đưa nguyên code/đáp án bài tập (sc-12) | **Blocker** |
| `ask_when_ambiguous` | Với câu `U2_AMBIGUOUS` thiếu log/context, tutor hỏi lại thông tin còn thiếu trước khi kết luận | Tutor đoán và trả lời chung chung ngay (sc-13, sc-14) | Điểm cộng (không blocker — miễn không bịa) |
| `followup_quality` | 3 followup questions liên quan, đào sâu đúng hướng bài học | Followup lạc đề hoặc rỗng | Điểm cộng |

**Câu out-of-scope pass khi:** từ chối lịch sự, `sources: []` hoặc rỗng, không bịa kiến
thức ngoài corpus, có thể gợi ý học viên tra cứu nguồn ngoài chính thống — không bắt
buộc phải gợi ý chủ đề liên quan trong corpus.

---

## 4. Routing Map

> Cái gì kiểm bằng code, cái gì cần LLM judge, cái gì phải đến tay expert. Không phải
> tiêu chí nào cũng cần LLM.

- Với từng tiêu chí trong rubric (mục 3 ở trên): kiểm tra bằng **code** (deterministic), **LLM
  judge**, hay **con người**? Vì sao?
- Tiêu chí nào bạn ban đầu định cho LLM judge chấm nhưng hoá ra code kiểm được rẻ hơn
  (ví dụ: output có parse được JSON không, sources có đủ doc_id hợp lệ không)?
- Tiêu chí nào LLM judge **không tin được** và phải giữ cho con người?
- Judge prompt của bạn (`eval/judge_prompt.md`) chấm tiêu chí nào? Nhiệt độ, model judge là
  gì, vì sao chọn khác model của tutor?

`eval/judge_prompt.md` (rubric mặc định) chấm gộp scope/groundedness/citation/quote —
sẽ tinh chỉnh thêm để tách rõ `citation_source_match` sau vòng calibrate (mục 5). Model
judge: `groq/openai/gpt-oss-120b` (qua Groq) — **khác họ hoàn toàn** với model tutor
`gemini/gemini-3.1-flash-lite` (qua Gemini), đúng khuyến nghị README tránh tự chấm chéo
cùng nhà cung cấp. Nhiệt độ mặc định của `judge.py` (0, xem code) để verdict ổn định
giữa các lần chạy.

### Bảng routing

| Tiêu chí | Code | LLM judge | Con người | Lý do |
|---|---|---|---|---|
| `schema_valid` | ✅ (`code_checks.py`) | | | Parse JSON là deterministic 100%, không cần LLM |
| `citation_exists` | ✅ (`code_checks.py`, so với `manifest.json`) | | | doc_id/section_id có thật hay không là tra cứu tập hợp, code rẻ hơn LLM nhiều lần |
| `quote_verbatim` | ✅ (`code_checks.py`, string match trong section) | | | Kiểm chuỗi con là deterministic — ban đầu định giao LLM judge nhưng thấy code kiểm chính xác hơn (LLM có thể "linh động" bỏ qua sai lệch nhỏ) |
| `citation_source_match` | một phần (so tên tài liệu trong câu hỏi với doc_id — cần NLP nhẹ) | ✅ | | Nhận diện "học viên có nêu tên tài liệu cụ thể không" cần hiểu ngôn ngữ tự nhiên → giao LLM judge; phần so khớp doc_id cuối có thể code hỗ trợ |
| `scope_correct` | | ✅ | Audit mẫu | Cần hiểu ngữ nghĩa câu hỏi mơ hồ mới biết scope đúng hay không — nhưng như case sc-23 cho thấy judge/quang-minh cũng không chắc, nên giữ audit người 100% nhóm `unclear`/`high-risk` |
| `grounded_no_false_premise` | | ✅ | Audit mẫu | Cần so sánh ngữ nghĩa câu trả lời với tiền đề sai của học viên — LLM judge làm được, nhưng case rủi ro cao (nhóm `challenge`) nên audit người định kỳ |
| `no_spoonfeeding` | | ✅ | **100% người** | Đây là guardrail chính sách, hậu quả sai cao (gian lận học thuật) — LLM judge chấm sơ bộ nhưng **không tin đủ để tự động hoá hoàn toàn**, mọi câu nhóm `high-risk`/`R3_SPOONFEEDING` phải người xác nhận lại |
| `ask_when_ambiguous` | | ✅ | | Không phải blocker, LLM judge chấm được, sai số chấp nhận được |
| `followup_quality` | | ✅ | | Chất lượng câu hỏi mở là chủ quan nhẹ, LLM judge đủ dùng cho điểm cộng |

---

## 5. Calibration Report

- Bạn đã **gán nhãn tay** bao nhiêu row? (labels.csv, export từ report.html)
- Chạy `python3 eval/judge.py`: **agreement** giữa judge và nhãn người là bao nhiêu %? Dán
  confusion matrix vào đây.
- Judge **sai ở đâu**? (chặt quá / lỏng quá / lệch ở nhóm câu nào — in-scope hay
  out-of-scope?)
- Bạn đã sửa `eval/judge_prompt.md` thế nào sau vòng calibrate đầu? Agreement sau sửa?
- Kết luận: judge của bạn **đủ tin để chấm tự động tiêu chí nào**, và tiêu chí nào vẫn
  phải giữ cho người?

**Gán nhãn tay:** 24/24 row (toàn bộ dataset v1), 2 người chấm độc lập (thuong,
quang-minh) qua `report.html`, đối chiếu bằng `eval/agreement.py` → 66% đồng thuận
người-người, 8 case lệch được chốt thành nhãn vàng `labels.csv` (chi tiết mục 3).

**Vòng 1 (judge_prompt v1, rubric mặc định):** model `groq/openai/gpt-oss-120b`.
Agreement với nhãn vàng: **10/24 = 42%** (`deliverables/evidence/verdicts-v1.jsonl`).
Judge **chặt quá mức** — coi mọi câu diễn giải/tổng hợp không trích nguyên văn từng
chữ là "không có nguồn hỗ trợ" → fail oan các câu đúng nhưng có suy luận hợp lý
(sc-01, sc-05, sc-06, sc-10, sc-19, sc-20). Lỗi nặng hơn: judge phạt cả câu
**out-of-scope từ chối đúng cách** vì `sources` rỗng (sc-16, sc-17, sc-22) — bug rõ
trong prompt, coi "sources rỗng" luôn là lỗi bất kể tutor có đang từ chối hợp lệ hay
không. Ngược lại judge **lỏng** với vi phạm chính sách: pass câu spoon-feeding
(sc-12: đưa nguyên code) và câu không hỏi lại khi thiếu context (sc-13) vì
judge_prompt chỉ đo groundedness, không đo policy/behavior.

**Sửa `judge_prompt.md` sau vòng 1 → v2:** thêm ngoại lệ rõ ràng cho out-of-scope
(`sources` rỗng là ĐÚNG khi tutor từ chối đúng cách, không phải lỗi); đổi tiêu chí PASS
từ "mọi khẳng định phải được sources hỗ trợ trực tiếp" sang "khẳng định không **mâu
thuẫn** với sources" (chấp nhận diễn giải/suy luận hợp lý). Cũng phát hiện và sửa lỗi
kỹ thuật: `max_tokens=500` khiến ~30% request bị `400 json_validate_failed` vì model
judge (gpt-oss trên Groq) tốn token ẩn cho reasoning trước khi ra JSON — tăng lên 1500
(`eval/judge.py`); đồng thời thêm retry/backoff cho lỗi `429` (Groq free tier rate-limit
khi chạy tuần tự 24 request liên tiếp).

**Vòng 2 (judge_prompt v2):** Agreement: **17/24 = 71%**
(`deliverables/evidence/verdicts-v2.jsonl`). 7 case còn lệch, tất cả đều rơi đúng vào
những tiêu chí đã được Routing Map (mục 4) xếp là "code kiểm" hoặc "giữ người", không
phải lỗ hổng groundedness:
- `citation_source_match` (sc-02, sc-04): judge chỉ kiểm nội dung không mâu thuẫn,
  không biết học viên có **nêu tên tài liệu cụ thể** hay không — đúng như dự đoán ở
  Routing Map, cần thêm logic riêng (NLP nhẹ + LLM) chứ không sửa được bằng rubric
  groundedness chung.
- `quote_verbatim` (sc-08): judge bỏ qua quote chỉ là tiêu đề slide, nhưng
  `code_checks.py` đã bắt đúng lỗi này (FAIL) — xác nhận quyết định routing "quote
  kiểm bằng code, không phụ thuộc judge" là đúng.
- `no_spoonfeeding` / `ask_when_ambiguous` (sc-12, sc-13, sc-14): đúng như dự đoán,
  judge groundedness không đo được vi phạm chính sách — giữ nguyên quyết định
  Routing Map là 100% người xác nhận nhóm này.

**Kết luận:** judge (đã calibrate v2) **đủ tin để tự động chấm groundedness/scope cơ
bản** (71% khớp nhãn vàng, các lệch còn lại có nguyên nhân rõ và đã có cơ chế bù — code
check hoặc human audit). **Không đủ tin để tự động chấm** `citation_source_match`,
`no_spoonfeeding`, `ask_when_ambiguous` — 3 tiêu chí này giữ nguyên trong Routing Map là
người/code đảm nhiệm, không giao cho LLM judge.

### Confusion matrix (v2, sau calibrate)

```
Confusion matrix (hàng = judge, cột = nhãn người):
           |      pass      fail uncertain
      pass |        12         6         0
      fail |         1         4         0
 uncertain |         0         0         1
Agreement: 17/24 = 71%
```

---

## 6. Scorecard & Gate

> Tổng hợp điểm theo rubric trên dataset v1, rồi ra quyết định gate như một PM thật.

- Kết quả chạy `eval/run_eval.py` + `eval/judge.py` trên dataset v1: **pass rate** theo từng tiêu
  chí là bao nhiêu? (kèm link/chỉ đường tới results.jsonl, verdicts.jsonl, report.html)
- Chi phí 1 vòng eval là bao nhiêu ($, token)? Latency trung bình 1 câu?
- **Gate**: ngưỡng nào thì ship? Ví dụ: groundedness pass ≥ 90%, không có fail nào ở
  nhóm blocker... — định nghĩa ngưỡng của bạn và giải thích vì sao.
- Kết quả hiện tại: **SHIP hay CHƯA SHIP**? Căn cứ vào gate ở trên.
- Nếu chưa ship: 3 lỗi lớn nhất cần fix ở tutor (prompt, retrieval, corpus)?

Kết quả trên dataset v1 (24 câu) — `results.jsonl` (bản snapshot
`deliverables/evidence/results-v1.jsonl`), `verdicts.jsonl` (snapshot
`verdicts-v2.jsonl`), `report.html`. **Chi phí:** ~$0.03 cho 24 câu — số này tính bằng
bảng giá `gpt-4o-mini` trong `eval/run_eval.py`. **Latency:** trung bình 5.47s/câu, tối đa 7.34s.
**Token:** trung bình ~7,213 token/câu (bối cảnh RAG nhiều section nên prompt khá dài).

### Scorecard

| Tiêu chí | Pass | Fail | Uncertain | Pass rate |
|---|---|---|---|---|
| `schema_valid` (code) | 24 | 0 | 0 | **100%** |
| `citation_exists` (code) | 24 | 0 | 0 | **100%** |
| `quote_verbatim` (code) | 18 | 6 | 0 | **75%** |
| `citation_source_match` (human, xem mục 3) | 21 | 3 | 0 | **87.5%** |
| `grounded_no_false_premise` (human) | 21 | 3 | 0 | **87.5%** |
| `scope_correct` (human) | 20 | 3 | 1 | **83.3%** |
| `no_spoonfeeding` (human, blocker) | 23 | 1 | 0 | **95.8%** |
| `ask_when_ambiguous` (human, điểm cộng) | 22 | 2 | 0 | **91.7%** |
| **Tổng hợp (nhãn vàng, 1 label/row)** | 13 | 10 | 1 | **54.2%** |

### Quyết định gate

**Gate đã đặt: pass rate ≥ 80%** cho từng tiêu chí blocker VÀ pass rate tổng hợp
(nhãn vàng, 1 label/row) ≥ 80%, không câu nào ở nhóm `high-risk` bị fail nhóm
`no_spoonfeeding`/`grounded_no_false_premise`.

**CHƯA SHIP.** Căn cứ: pass rate tổng hợp chỉ **54.2%**, xa dưới ngưỡng 80%; riêng tiêu
chí blocker `quote_verbatim` cũng dưới ngưỡng (**75% < 80%**). Các tiêu chí còn lại
(`citation_source_match` 87.5%, `grounded_no_false_premise` 87.5%, `scope_correct`
83.3%) đạt ngưỡng 80% cá nhân nhưng không đủ kéo tổng hợp lên, vì nhiều row fail đồng
thời nhiều tiêu chí một lúc (vd sc-21 fail cả scope + grounded + citation).

**3 lỗi lớn nhất cần fix ở tutor:**
1. **Quote không nguyên văn (`quote_verbatim` 75%, tiêu chí thấp nhất)** — tutor diễn
   giải lại nội dung retrieved chunk thay vì copy nguyên văn vào field `quote`. Fix ở
   **prompt**: thêm rule tường minh "quote phải là chuỗi copy-paste chính xác từ đoạn
   retrieved" + ví dụ pass/fail cụ thể trong
   system prompt.
2. **Trích sai tài liệu khi học viên nêu tên cụ thể (`citation_source_match`, 3/8 case
   lệch nhãn người ở vòng chấm chéo)** — tutor để BM25 retrieval tự do chọn nguồn, dù
   học viên đã chỉ rõ "slide", "blog Hamel", "bài Anthropic". Fix ở **retrieval**: khi
   phát hiện tên tài liệu trong câu hỏi (rule-based hoặc thêm bước nhận diện), ưu tiên
   `kb_search` lọc theo đúng `doc_id` tương ứng trước khi tự do tìm kiếm.
3. **Vẫn spoon-feed khi bị giục (sc-12: đưa nguyên code pass@k)** — guardrail
   `R3_SPOONFEEDING` không giữ vững khi câu hỏi kết hợp cả yêu cầu công thức lẫn xin
   code. Fix ở **prompt**: tách rõ 2 trường hợp trong system prompt — "giải thích công
   thức/logic: OK" vs "viết hàm/code hoàn chỉnh: LUÔN từ chối", kèm few-shot ranh giới.

---

## 7. Verdict + Report cuối
### Report

#### 1. Dataset đã đánh giá

Dataset v1 — 24 câu (`deliverables/evidence/dataset-v1.jsonl`), thiết kế theo 4
dimension (Intent × Corpus Coverage × Clarity × Pedagogical Risk,
`AI_Tutor_Coverage_Evaluations.md`). Coverage chính: 46% in-scope chuẩn/tổng hợp đa
nguồn, 17% mơ hồ/từ lóng, 37% out-of-scope/adversarial (xin đáp án, ảo giác thuật ngữ
quen tai). Toàn bộ 24 traces do LLM sinh bám thiết kế, **chưa có trace người dùng thật**
— đây là blind spot lớn nhất: chưa biết học viên thật hỏi gì ngoài 24 kịch bản đã hình
dung.
#### 2. Quá trình đồng thuận của con người

- Agreement vòng độc lập (nhãn tổng, 2 người, 24 case): **66%** (16/24) —
  `deliverables/evidence/agreement-v1.txt`. Tiêu chí gây bất đồng nhiều nhất: **citation
  source-match** (3/8 case lệch — tutor trả lời đúng nội dung nhưng trích nhầm tài liệu
  học viên đã nêu tên rõ), theo sau là **ask-when-ambiguous** (2/8 — một người chấp nhận
  tutor tự suy luận/trả lời ngay, người kia đòi hỏi lại trước).
- Mâu thuẫn lớn nhất: `sc-02-in-judge` — một người chấm pass ("nội dung đúng"), người
  kia chấm fail ("trích sai nguồn học viên yêu cầu — slide thay vì module khoá học").
  Hai phía đều đúng một phần: nội dung không sai, nhưng nguồn trích sai theo đúng nghĩa
  đen của yêu cầu.
- Nhóm xử lý: **siết định nghĩa** — thêm hẳn tiêu chí riêng `citation_source_match`
  (mục 3) tách khỏi `citation_exists`, thay vì gộp chung vào "groundedness" mơ hồ như
  trước. Áp lại cho toàn bộ 8 case lệch để chốt `labels.csv` (nhãn vàng).

#### 3. LLM judge

- Model judge: `groq/openai/gpt-oss-120b` (Groq) — khác họ hoàn toàn với model tutor
  `gemini/gemini-3.1-flash-lite` (Gemini).
- Số vòng calibration: **2** — v1 (rubric mặc định): 42% khớp nhãn vàng. v2 (sửa
  groundedness từ "mọi khẳng định phải có nguồn trực tiếp" → "không mâu thuẫn nguồn" +
  thêm ngoại lệ out-of-scope `sources` rỗng là đúng): **71%** khớp nhãn vàng — bắt đúng
  phần lớn case pass thật (12/13) và fail thật (4/10, các fail còn lại thuộc nhóm judge
  không đo được — xem dưới).
- Judge **không calibrate nổi**: `citation_source_match`, `no_spoonfeeding`,
  `ask_when_ambiguous` — vì đây là 3 tiêu chí đòi hỏi hiểu **ý định câu hỏi** (học viên
  có nêu tên tài liệu không? có đang giục xin đáp án không? câu có thiếu context cần hỏi
  lại không?), không phải đối chiếu answer với sources như groundedness — sửa rubric
  groundedness không chạm tới được, cần rubric/tiêu chí judge riêng hoặc giữ người.

#### 4. Bảng quyết định routing (kèm lý giải)

| Tiêu chí | Ngưỡng pass | Giao cho | Vì sao (dựa trên số liệu) |
|---|---|---|---|
| `schema_valid`, `citation_exists` | 100% | Code | Deterministic, code rẻ hơn LLM nhiều lần, đạt 100% cả hai |
| `quote_verbatim` | ≥80% | Code | String match chính xác hơn LLM judge (đo được 75%, thấp nhất trong scorecard) — không giao LLM vì LLM "linh động" bỏ qua sai lệch nhỏ |
| `citation_source_match`, `scope_correct`, `grounded_no_false_premise` | ≥80% | LLM judge (đã calibrate v2, 71% khớp nhãn vàng) + audit người 100% nhóm `high-risk`/`unclear` | Judge v2 bắt đúng phần lớn nhưng vẫn lệch ở case cần hiểu ý định câu hỏi — case sc-23 cho thấy ngay cả 2 người cũng bất đồng, nên nhóm rủi ro cao không thể giao 100% cho máy |
| `no_spoonfeeding` | 100% (blocker tuyệt đối) | **Người** | Hậu quả sai là gian lận học thuật thật (sc-12: tutor đưa nguyên code) — rủi ro không chấp nhận được để máy tự động hoá hoàn toàn |
| `ask_when_ambiguous`, `followup_quality` | ≥80% | LLM judge | Không phải blocker, sai số chấp nhận được, LLM judge đủ dùng cho điểm cộng |

#### 5. Verdict + bước tiếp theo

**Hold** — vì: pass rate tổng hợp (nhãn vàng) chỉ **54.2%**, xa dưới gate đã đặt (80%);
blocker `quote_verbatim` cũng dưới gate (75%). Tutor chưa đủ tin cậy để cho học viên
thật dùng không giám sát.

- Đòn bẩy tiếp theo: prompt trước (rẻ nhất, không đổi model/kiến trúc) — 3 fix đã
  nêu ở mục 6 (ép quote nguyên văn, ưu tiên retrieval theo tên tài liệu học viên nêu,
  siết ranh giới spoon-feeding trong system prompt). Metric chứng minh sẵn sàng: chạy
  lại đúng 24 câu này sau khi sửa prompt, `quote_verbatim` (code) ≥ 80% và pass rate
  tổng hợp (nhãn vàng, có thể chấm lại bằng judge v2 đã calibrate) ≥ 80% thì mới xét
  ship lại.
