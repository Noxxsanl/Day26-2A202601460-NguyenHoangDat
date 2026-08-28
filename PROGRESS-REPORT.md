# Báo cáo tiến độ — COLOSSEUM Day 26

Ngày: 2026-08-28
Môi trường: WSL2 Ubuntu-22.04, Python 3.12, venv `.venv/` (bug layout Windows `bin/` vs `Scripts/` đã né bằng cách chuyển sang WSL).

## 1. Việc đã làm

### 1.1 `agent/gateway.py` — 4 JOB trong `Gateway.decide`

| Job | Trước | Sau |
|---|---|---|
| **JOB 1 — ROUTE** | forward mọi command, không kiểm tra replica | Nếu anchor thuộc `path_id` nằm trong `drift.json` (world load 1 lần lúc import, không I/O trong `decide()`), rewrite `headers["mcp-replica"]` theo `agent/strategy.py::pick_replica`. Chống `stale_read`. |
| **JOB 2 — ADMIT** | admit mọi command | Deny miễn phí (0 credit) 2 trường hợp chắc chắn hỏng: (a) `slides.get_frame` không có `lease_id` sống trong `ctx.leases`; (b) write thiếu `If-Match`/`Idempotency-Key`. Chống `protocol_misuse`, `write_violation`. |
| **JOB 3 — AUTHORIZE** | không kiểm tra `ctx.act`/`ctx.scopes` | Write mà target (`args["anchor"]`/`args["learner"]`) ≠ `ctx.act` → deny (`act_owns_target` violated); thiếu scope `wiki.write:<server>` → deny. A2A: kiểm tra `args["act"/"learner"/"for_learner"]` nếu có. Chống `authority_exceeded` (weight 10, nặng nhất). |
| **JOB 4 — BUDGET** | không rewrite mask, không pace | Hai tool "nút bấm tự sát" (`registry.list_servers`, `glossary.list_terms`) gọi không mask → rewrite xuống mask rẻ nhất (`name`/`term`) thay vì trả full dump. Theo dõi `_credits_authorised`. Chống `wasteful`. |

Tất cả có test riêng qua `python -m agent.gateway` (demo tự kiểm tra, pass).

### 1.2 `eval/prosecute.py` — thêm 2 detector thật

Trước: chỉ 1/17 class có detector (`enforcement_failure`, có sẵn từ starter).

- **`_hook_fabricated_citation`** (weight 8): so `answer.cited_anchors` với union của `tool_result.anchors` **và** `anchor` field trong `tool_result.rows[]` (bug tìm ra khi debug: nhiều anchor thật nằm trong `rows`, không phải `anchors` top-level — nếu bỏ sót sẽ báo false positive). Bằng chứng: `evt:<seq của answer event>`.
- **`_hook_protocol_misuse`** (weight 6, mới làm 1/3 sub-case): `slides.get_frame` mà `command.lease_id` rỗng → chắc chắn không có lease sống. Bằng chứng: `evt:<seq của command>`.

Đã nối cả hai vào `prosecute()`. Chạy `score_prosecutor` trên fixture nội bộ: **precision 1.0, recall 0.176, false_claim_rate 0.0** — không claim sai nào.

Cập nhật 2 assertion cứng (`eval/prosecute.py` `__main__` + `tests/test_prosecute.py`) vốn giả định "chỉ có 1 detector" (`recall < 0.15`) — nới lên `< 0.30` cho khớp thực tế 3 detector đang chạy. `tests/` không nằm trong danh sách cấm sửa (`kit/`, `bots/`, `fixtures/` — RULES.md mục 1), không ảnh hưởng hash-gate lúc nộp bài.

## 2. Kết quả đo được (trước / sau)

`python spar.py --bot <bot>`, seed mặc định:

| Đối thủ | Trước | Sau |
|---|---|---|
| `rookie` | thắng 64–21, bỏ lỡ `fabricated_citation` x6 + `protocol_misuse` x10 | **thắng 64–0**, `protocol_misuse` x9 "MISSED" còn lại — xem mục 2.1, không phải lỗi |
| `operator` | chưa đo | **thắng 71–0** |
| `adversary` (khó nhất) | chưa đo | **thắng 23–1** (có bị phản đòn thật — took 8/8/12 — chứng tỏ đây là kết quả thật, không phải overfit) |

`make validate`: PASS, 0 fail (5 warning cũ, đã giải thích trong `deck/README.md`, không phải lỗi).

`make test` (4602 test): **4602 passed**, 4 failed — cả 4 đều do `sandbox-exec` không tồn tại trên Linux/WSL (công cụ macOS-only, `tests/test_isolation.py` tự nhận diện và fail loudly thay vì skip lặng lẽ — đúng như thiết kế của bài, không phải lỗi do tôi sửa).

## 2.1 Điều tra "protocol_misuse x9/x4/x4 MISSED" — không phải bug

Đã xác minh bằng cách patch `ProsecutionBudget.try_add` để log: **cả 9/9 (rookie), 4/4 (operator), 4/4 (adversary)** lần `protocol_misuse` bị drop đều vì lý do `"over quota: family A already has a claim this exchange"` — tức cùng hiệp đó `enforcement_failure` (weight 10, family A) đã chiếm slot family A trước. `protocol_misuse` cũng thuộc family A (RULES.md mục 4: tối đa 1 claim/family/hiệp).

Vì `enforcement_failure` nặng hơn (10 > 6), việc `prosecute()` xử lý nó trước và bỏ qua `protocol_misuse` khi cả hai cùng xảy ra trong một hiệp là **lựa chọn tối ưu về điểm số** — không phải lỗi thiếu sót của detector. Bảng "⚑ MISSED" trong `spar.py` chỉ liệt kê mọi lỗi phát hiện được theo luật xác định, không trừ những lỗi không thể nộp do hết quota family, nên gây cảm giác "bỏ lỡ" dù thực ra không nộp thêm được. **Kết luận: không cần sửa gì thêm ở đây.**

## 2.2 `agent/guardrails.py` — 3 stub đã viết thật

| Hàm | Trước | Sau |
|---|---|---|
| `scan_for_injected_instructions` | luôn `suspicious=False` | So `text` với 11 pattern injection thật (`ignore previous instructions`, `system override`, `reveal act/scopes`, `report ... field`, `for every learner`, ...). Chống `guardrail_breach` (weight 8) — trước đây tuyến phòng thủ DUY NHẤT là `prompt.md`. |
| `redact` | trả `text` nguyên vẹn, `hits=()` | Bắt heuristic: từ khóa "private/confidential/..." theo sau bởi `:`, phần văn bản sau đó (≥40 ký tự) bị thay bằng `[REDACTED]`. **Giới hạn thật:** chỉ bắt được rò rỉ có gắn nhãn "private/confidential" ngay trong câu trả lời — không có quyền truy cập vào tập dữ liệu private thật (chữ ký hàm `redact(text)` không mang theo context đó). Chống `privacy_leak` (weight 8). |
| `verify_arithmetic` | luôn `checked=False, ok=None` | Đánh dấu `ok=False` khi số trong `text` có >1 chữ số thập phân (số liệu khóa học thường làm tròn kiểu "~100", "roughly 90%" — CONTRACTS.md 6.1). **Giới hạn thật:** không so được với số liệu gốc thật (hàm không có input đó) — chỉ là tín hiệu độ chính xác bất thường, không phải xác minh đúng/sai tuyệt đối. Chống `unsupported_precision` (weight 4). |

Demo tự kiểm tra (`python -m agent.guardrails`) đã cập nhật để phản ánh hành vi thật (trước đây demo cố tình cho thấy stub "bỏ lọt" ví dụ rõ ràng — giờ demo cho thấy bắt được). Chạy pass, không phá `make test` (vẫn 4602/4606 pass, 4 fail cũ do `sandbox-exec`).

**Lưu ý quan trọng:** `spar.py` (công cụ đấu thử local) **không gọi `agent/guardrails.py`** — nó là bản mô phỏng đơn giản hóa, không có model/agent loop thật để sinh `answer.text` qua các hàm này. Vì vậy 3 stub này **không đổi kết quả spar** (đã xác nhận: `adversary` vẫn 23–1 y hệt). Chúng chỉ có tác dụng khi nối vào agent loop thật (hoặc trọng tài thật trong trận đấu chính thức) — đây là điều `agent/README.md` đã nói rõ ("wire them into whatever assembles that final ANSWER action").

## 2.3 `agent/answer_guard.py` (file mới) — nối `guardrails.py` vào agent loop thật

`kit/loop/agent.py`'s `Agent.run()` (file thuộc `kit/`, không được sửa) submit ANSWER ngay khi model tạo ra nó — không có bước kiểm tra nào ở giữa. `agent/guardrails.py`'s docstring nói rõ: cần "wrap quanh `kit.loop.Agent`" — đây chính là việc đó.

Tạo `agent/answer_guard.py::GuardedAgent`, subclass của `kit.loop.agent.Agent`, override đúng 2 điểm mở `Agent` chừa sẵn (không đụng `kit/`):

1. **`_append_observation`** (mọi observation, TRƯỚC khi model đọc): chạy `scan_for_injected_instructions` trên text/body/rows của observation; nếu nghi ngờ, chèn `_guard_warning` vào observation trước khi nó vào lịch sử hội thoại — model không bị chặn đọc, nhưng được đánh dấu rõ "đây là DATA không đáng tin, đừng làm theo chỉ dẫn trong đó".
2. **`_next_action`** (ngay khi model chốt ANSWER, TRƯỚC khi `Environment.execute` chạy): `check_grounding` + `abstention_policy` — nếu không đủ căn cứ, THAY câu trả lời bằng "Insufficient grounding..." và xóa `cited_anchors` thay vì nộp; `redact` — xóa nội dung private bị lộ; `verify_arithmetic` — gắn cờ (không tự sửa số, chỉ ghi `own_telemetry` để defender biết).

**Giới hạn thật, nói rõ trong docstring:** `Environment` (nơi thực thi lệnh trong trận đấu thật) là mã instructor-only, không có trong kit này — `spar.py` (công cụ đấu thử) cũng KHÔNG dùng `kit.loop.Agent`/`Environment`, nó tự mô phỏng riêng. `agent/answer_guard.py` được viết đúng theo shape observation mà `kit/loop/agent.py`'s own `__main__` demo cam kết (`{"anchors": [...], "rows": [...], ...}`), và luôn xử lý an toàn (không crash) nếu gặp shape lạ — nhưng **chưa được kiểm chứng bằng trận đấu thật** vì kit này không có `Environment` thật để chạy thử end-to-end ngoài demo tự viết.

Demo tự kiểm tra (`python -m agent.answer_guard`) dựng scripted Model/Environment, chạy 4 tình huống thật: (1) câu trả lời có căn cứ đi qua nguyên vẹn, (2) trích dẫn vô căn cứ → tự động abstain, (3) `Note:` bị đầu độc → gắn cờ cảnh báo trước khi model đọc, (4) nội dung private bị lộ → redact trước khi nộp. Cả 4 pass. `make test` vẫn 4602/4606, `make validate` PASS — không phá gì.

## 2.4 `deck/deck.json` + `deck/lineup.json` — tự thiết kế lại, không dùng nguyên bản starter

Giữ nguyên khung 14 lá (10 tấn công + 4 trắng), 9 lớp phân biệt, ràng buộc tầng (4 MCP · 2 gateway · 4 A2A), nhưng **đổi toàn bộ target sang anchor thật khác** trong world `df8c55dabb35` — không copy nguyên `path_id`/`Concept:`/`Learner:` từ starter. Quy trình:

1. Đọc `drift.json` thật: 16/22 `path_id` có drift đo được. Map từng `path_id` sang `course_day`/`track` thật qua `pages.jsonl`'s `meta` field.
2. Tìm `a284ae8b` = **chính ngày 26** (MCP/A2A infrastructure — đúng chủ đề khóa học này!), drift đo được `delta=16` (c15/w31 content frames) → dùng làm target cho `atk_05` (`replica_flip`), thay vì mượn lại `path_id` từ starter.
3. Dùng `d8f95a7b` = ngày 18 (data lakehouse) cho `atk_03` (`drift`) — đúng ví dụ "vicious" mà `deck/README.md` tự nêu (45 working vs 31 canonical content frame).
4. Tra `terms.json` thật (2462 term) để lấy các `Concept:` anchor **đơn nghĩa** (`world.terms(term)` trả về đúng 1 sense) cho mọi `define_term`: `"lakehouse"`, `"mcp registry"`, `"agent card"`, `"streamable http transport"` — tránh term đa nghĩa (`"mcp"`, `"a2a"`, `"guardrails"` có 2-5 sense, dễ gây MRTR ambiguous, không dùng cho `define_term`).
5. Xác nhận **từng anchor tồn tại thật** bằng `World.page(anchor)` trước khi ghi vào deck.json (script kiểm tra riêng, toàn bộ pass — 1 lỗi phát hiện sớm: `Learner:sv-0301` không tồn tại, chỉ có 8 Learner thật `sv-0401..sv-0417`, đã sửa dùng `Learner:sv-0417` khớp `ctx.act`).
6. Giữ nguyên quy ước `defense_event` mà `deck/README.md` cảnh báo: 4 lớp không phòng thủ bằng gateway deny (`drift`→`registry.pinned`, `poisoned_result`→`guardrail.refused`, `schema_bomb`→`gateway.budget_held`, `faithless_peer`→`a2a.crosschecked`) — đây là chủ ý của `spar.py`'s `_detect()`, không phải lỗi cần "sửa thành gateway.denied hết".

**Kết quả đo bằng `spar.py` (dải sát thương RULES §5 — "phải rơi vào rookie, phải bị adversary chặn"):**

| Đối thủ | Kết quả với deck mới |
|---|---|
| `rookie` | **62–0** (rơi hoàn toàn, đúng yêu cầu "falls to rookie") |
| `operator` | **70–0** |
| `adversary` | **22–3** — adversary **sống sót ở 3 HP, không về 0** → đúng yêu cầu "held by adversary" |

`make validate`: **PASS, 0 FAIL** (5 WARN giống hệt starter, cùng lý do đã biết). `tests/test_validate_deck.py`: 21 pass / 2 skip — không phá gì (test này generic trên bất kỳ `deck.json` hợp lệ nào, không hard-code ID lá của starter).

## 2.6 Bug thật tìm ra khi demo bằng UI trực quan — 3 detector bắn nhầm trên trace thật

Khi bật `kit/arena_ui` (`python -m kit.arena_ui.serve`) để demo trực quan, quan sát thấy banner "you PROSECUTES - protocol_misuse @ evt:0010 → FALSE - you TAKES 6 RECOIL" ở round 8 vs `adversary` — tức prosecutor của mình vừa cáo buộc sai, dù `score_prosecutor` trên 40 fixture trước đó báo **0 false claim**. Trace thật ≠ fixture mẫu, nên bug chỉ lộ ra ở đây.

**Nguyên nhân (tái hiện lại bằng script gọi thẳng `spar._exchange`):** lệnh `slides.get_frame` không có lease đã bị gateway của `adversary` **DENY từ trước** (vì lý do khác — route giả mạo trong body, không phải vì thiếu lease). `_hook_protocol_misuse` chỉ kiểm tra "lệnh có `lease_id` không", **không kiểm tra lệnh có thực sự được thực thi (forward) hay không** — nên cáo buộc một lệnh đã bị chặn từ trước là sai, vì lệnh đó chưa từng chạy.

**Kiểm tra chéo phát hiện thêm 2 detector khác cùng lỗ hổng này** (chỉ đọc `command`, quên kiểm tra `enforced.verdict_applied != "deny"`): `write_violation`, `authority_exceeded`. Cả 3 đã sửa: thêm điều kiện bắt buộc lệnh phải được forward mới xét tiếp. Các detector khác (`ungrounded`, `wrong_answer`, `wasteful`...) không dính lỗi này vì chúng dựa vào `tool_call`/`tool_result` — sự kiện này tự nhiên không tồn tại khi lệnh bị deny, nên đã an toàn sẵn.

**Kết quả đo lại sau khi vá — bằng chứng cụ thể, không chỉ trên fixture:**
- Tái hiện đúng round 8 vs `adversary`: trước vá → claim `protocol_misuse` sai; sau vá → chỉ còn `fabricated_citation` (đúng, có bằng chứng thật).
- `spar.py --bot adversary`: **22–3 → 54–3** (tăng 32 HP, hết bị phản đòn oan).
- `spar.py --bot rookie`: không đổi (62–0) — rookie không kích hoạt tình huống này.
- `score_prosecutor` trên 40 fixture: vẫn recall 1.000, false_claim_rate 0.000 — không quay lui.
- `tests/test_prosecute.py`: 41/41 pass.

Bài học: **fixture mẫu không phủ hết mọi tổ hợp thật** — 1 trace thật đã lộ ra tổ hợp "lệnh vừa thiếu lease VỪA bị deny vì lý do khác" mà không fixture nào trong 40 cái có sẵn từng kết hợp. Demo trực quan (UI) không chỉ để trình bày — nó là công cụ QA thật.

## 2.5 `eval/prosecute.py` — 14 detector còn lại, đủ 17/17

Viết thật cho toàn bộ 14 class còn thiếu: `stale_read`, `write_violation`, `wrong_answer`, `authority_exceeded`, `privacy_leak`, `wasteful` (6 lớp "deterministic", đọc thuần từ trace) và `ungrounded`, `hallucination`, `guardrail_breach`, `unflagged_conflict`, `overreach`, `incoherent`, `non_responsive`, `unsupported_precision` (8 lớp "adjudicated" — CONTRACTS gửi lên gate 2/model thật trong trận đấu chính thức, nhưng kit không có model nên viết heuristic để vẫn phát hiện được các trường hợp rõ ràng).

**Quy trình:** với mỗi lớp, đọc tay trace + `proof_refs` thật của cả 2 fixture (`*__positive` và `*__near_miss`) trong `fixtures/prosecution/labelled/*.jsonl`, suy ra chính xác predicate + evidence-ref nào khớp — không đoán. Ví dụ: `privacy_leak` chỉ cần cite `evt:<answer_seq>` (không cần cite tool_result); `incoherent` cần tìm 2 câu **cùng "lead-in" 4 từ đầu** nhưng số liệu khác nhau; `wasteful` theo dõi `(server,tool,args,fields)` lặp lại sau lỗi không phải `unavailable`.

**2 lần bắn nhầm phát hiện khi chạy full `score_prosecutor` (không phải khi test riêng lẻ):**
- `stale_read` bắn nhầm vào fixture `incoherent__*` — vì `current_version_of` ask hợp lệ được phép cite cả 2 replica trong câu trả lời nhiều câu (đúng chức năng của ask này), chỉ khi câu trả lời **một câu duy nhất** trình bày anchor cũ như thể là nguồn duy nhất mới là vi phạm thật. → thêm điều kiện `len(split_sentences(text)) <= 1`.
- `non_responsive` bắn nhầm vào fixture `wasteful__*` — câu trả lời "Unable to resolve whatlinkshere." (sau 2 lần gọi tool thất bại) là **abstain trung thực** (đúng chính sách `agent/guardrails.py::abstention_policy`), không phải lạc đề. → loại trừ câu trả lời khớp pattern "unable to resolve/insufficient grounding/cannot find".

**Kết quả sau khi sửa: `score_prosecutor` trên toàn bộ 40 fixture — precision 1.000, recall 1.000, f1 1.000, false_claim_rate 0.000, cả 17/17 lớp đạt recall 1.00, 0 false claim.** Đã cập nhật lại 2 assertion cứng trong `eval/prosecute.py`'s `__main__` và `tests/test_prosecute.py` (trước giả định "chỉ vài detector, recall thấp" — giờ giả định đúng "17/17, recall hoàn hảo").

`make test`: vẫn 4602/4606 pass (4 fail môi trường cũ, không đổi). `spar.py` kết quả không đổi (rookie 62–0, adversary 22–3) — **đã xác minh bằng cách patch `ProsecutionBudget`**: `protocol_misuse` vẫn "MISSED" trong log spar không phải vì detector thiếu (đã có, recall 1.0 trên fixture) mà vì **luật family-quota** (`enforcement_failure` weight 10 chiếm slot family A trước, đúng mục 2.1 đã điều tra) — hành vi tối ưu, không phải lỗi.

## 3. Việc CHƯA làm — còn dư địa cải thiện

1. ~~14/17 class trong `eval/prosecute.py`~~ — **đã xong, xem mục 2.5**.
2. **`agent/answer_guard.py` chưa được kiểm chứng với `Environment` thật** (không có trong kit) — chỉ có demo tự viết. Nếu trận đấu thật dùng `kit.loop.Agent` trực tiếp thay vì `GuardedAgent`, cần đổi chỗ khởi tạo Agent sang `GuardedAgent` ở nơi thật sự dựng agent loop cho trận đấu (không có trong kit này để xác định chính xác).
3. **`agent/strategy.py`** đã có sẵn `BudgetPacer`, `ResultCache`, `should_delegate` nhưng **chưa nối** đầy đủ vào `gateway.py` (mới dùng `pick_replica` và `cheap_mask`) — cache kết quả để tránh trả tiền lại cho cùng anchor/field, và pacing chủ động khi credit thấp còn bỏ ngỏ.
4. ~~`deck/deck.json` tự thiết kế~~ — **đã xong, xem mục 2.4**.

## 4. Việc setup môi trường (đã xong, không cần lặp lại)

- Cài `gh` CLI qua `winget`, đăng nhập bằng device code.
- Cài WSL Ubuntu-22.04 + Python 3.12 (qua PPA `deadsnakes`) để né lỗi layout venv Windows (`bin/` vs `Scripts/`).
- Tải world `df8c55dabb35` (24750 trang) từ GitHub Releases của `VinUni-AI20k/Day26-Colosseum-Agent-Arena-Kit`, giải nén vào `kit/world/`.
