# Design Doc — Agent Eval Playground

> Mục tiêu: Biến việc "đánh giá prompt / skill / agent-workflow" từ **cảm tính** thành **có số liệu**, lặp lại được và (phần lớn) tự động hoá.

---

## 0. TL;DR

Team đang học prompt / set-up agents / workflow và liên tục tải các prompt-pack, skill, agent-workflow từ GitHub về thử. Vấn đề: không biết cái nào tốt, đánh giá mất thời gian và chủ quan.

**Giải pháp:** Một **eval playground** — bộ task chuẩn hoá, cố định, snapshot sạch. Mỗi candidate (prompt/skill/workflow) chạy qua cùng bộ task → auto-grade (test + LLM-judge + cost) → leaderboard so sánh. Đây là cách thu nhỏ của SWE-bench / Terminal-Bench / inspect-ai cho quy mô team.

**Nguyên tắc cốt lõi:** mỗi task phải có **ground truth kiểm chứng được bằng máy**.

---

## 1. Mục tiêu & Phi mục tiêu

### Mục tiêu
- Chạy 1 lệnh → ra bảng so sánh các candidate trên cùng bộ task.
- Số liệu khách quan (test pass rate, cost) làm xương sống; LLM-judge cho phần chất lượng mở.
- Lặp lại được: cùng candidate + cùng task → kết quả ổn định (trong giới hạn variance).
- Dễ thêm task mới và thêm candidate mới.

### Phi mục tiêu (giai đoạn đầu)
- Không cố benchmark "app hoàn chỉnh từ raw idea" (T4) ngay — để sau khi nền T1–T3 vững.
- Không xây UI cầu kỳ — leaderboard dạng bảng/markdown là đủ.
- Không tự viết harness từ 0 nếu fork được (xem §8).

---

## 2. Khái niệm

| Thuật ngữ | Nghĩa |
|-----------|-------|
| **Task** | Một bài toán cụ thể, cố định, có ground truth. VD "fix null-pointer bug trong repo X". |
| **Candidate** | Thứ đang được đánh giá: 1 prompt, 1 skill, 1 agent-workflow, hoặc 1 cấu hình model. |
| **Run** | 1 lần candidate chạy qua 1 task trên 1 snapshot sạch. |
| **Grade** | Điểm của 1 run, gồm objective + judge + cost. |
| **Leaderboard** | Bảng tổng hợp điểm trung bình của các candidate trên toàn bộ task-set. |

---

## 3. Phân tầng task (Task Taxonomy)

Tách theo độ khó & độ dễ chấm. Bắt đầu từ tầng dễ chấm.

| Tier | Loại | Ví dụ | Ground truth | Độ dễ auto-eval |
|------|------|-------|--------------|-----------------|
| **T1** | Fix bug có sẵn | Repo có bug + test fail sẵn | Hidden test pass | ⭐⭐⭐ |
| **T2** | Viết 1 function theo spec | Implement theo docstring | Hidden test pass | ⭐⭐⭐ |
| **T3** | Thêm feature vào code có sẵn | Thêm endpoint/CLI flag | Acceptance test | ⭐⭐ |
| **T4** | Raw idea → app hoàn chỉnh | Idea→plan→impl→test→ship | Rubric + smoke test | ⭐ |

**Lộ trình:** xây vững T1–T2 trước (5–10 task), rồi T3, cuối cùng mới T4.

---

## 4. Cấu trúc một Task

Mỗi task = 1 thư mục cố định:

```
tasks/t1-fix-null-pointer/
├── meta.yaml          # id, tier, ngôn ngữ, tags, mô tả ngắn
├── prompt.md          # đề bài đưa cho agent (KHÔNG chứa lời giải/hidden test)
├── repo/              # snapshot code ban đầu (git pinned commit)
├── tests/             # hidden test suite — KHÔNG đưa cho agent
├── rubric.yaml        # tiêu chí chấm mở (chủ yếu cho T3/T4)
├── setup.sh           # cài deps, dựng môi trường
├── verify.sh          # chạy tests/, trả exit code + JSON kết quả
└── solution.ref/      # (optional) lời giải tham chiếu để pairwise-compare
```

### meta.yaml (ví dụ)
```yaml
id: t1-fix-null-pointer
tier: T1
language: python
tags: [bugfix, null-safety]
description: "API crash khi user profile thiếu field avatar"
timeout_sec: 600
max_tool_calls: 50
```

### Nguyên tắc thiết kế task
1. **Hidden test không bao giờ lộ** cho agent — chỉ dùng ở bước grade.
2. **Snapshot reset sạch tuyệt đối** mỗi run (git checkout / container mới). Đây là lỗi phổ biến nhất phá hỏng benchmark.
3. **Tránh contamination**: đừng dùng task quá nổi tiếng (lời giải đầy trên mạng → agent "thuộc bài").
4. **Đề bài rõ ràng, đóng**: với T1–T3, spec phải đủ chặt để test xác định đúng/sai.

---

## 5. Evaluation — Cách chấm (phần khó nhất)

Không có 1 con số duy nhất. Chấm **đa tầng**:

### Tầng A — Objective metrics (auto 100%, xương sống)
Dùng cho T1–T3, là tín hiệu mạnh nhất.
- **Test pass rate**: % hidden test xanh.
- **Build/lint pass**: code chạy được, không lỗi cú pháp.
- **Efficiency/cost**: token tiêu thụ, số tool-call, wall-clock, số lần edit.

### Tầng B — LLM-as-judge (auto, cho phần chất lượng mở)
Dùng khi không viết test được ("code có sạch không", "UX ổn không"):
- **Rubric cố định** + model mạnh (Opus) chấm theo từng tiêu chí.
- **Chống thiên vị:**
  - *Blind grading*: giấu candidate nào tạo ra output.
  - *Pairwise comparison* ("A vs B, cái nào tốt hơn") đáng tin hơn chấm điểm tuyệt đối.
  - *Panel/multi-sample*: nhiều judge hoặc nhiều lượt → lấy đa số.

### Tầng C — Human spot-check (thủ công, ít, để calibrate)
LLM-judge có thể sai hệ thống. Định kỳ người chấm tay ~10% mẫu → kiểm tra judge có lệch không. Khớp người → tin & scale.

### Công thức tổng (trọng số do team chỉnh)
```
Score = w1·test_pass + w2·judge_quality − w3·cost_normalized
```

### Xử lý variance (BẮT BUỘC)
LLM không deterministic. Phải:
- Chạy **mỗi (candidate, task) ≥ 3 lần**.
- Báo cáo **trung bình + pass@k + độ lệch chuẩn**, không tin 1 lần chạy.

---

## 6. Vòng lặp tự động hoá

```
┌────────────────────────────────────────────────────────────┐
│ 1. Lấy candidate (prompt/skill/workflow từ GitHub/local)     │
│ 2. Với MỖI task × N lần lặp:                                 │
│      a. Reset snapshot sạch (git/container)                  │
│      b. setup.sh → dựng môi trường                           │
│      c. Chạy candidate trên prompt.md (đo token/tool/time)   │
│      d. verify.sh → chạy hidden tests, thu JSON              │
│      e. LLM-judge theo rubric (blind)                        │
│ 3. Tổng hợp Grade → ghi vào kết quả                          │
│ 4. Render leaderboard (markdown/CSV)                         │
│ 5. Human calibrate định kỳ (~10% mẫu)                        │
└────────────────────────────────────────────────────────────┘
```

Bước 2–4 script hoá hoàn toàn → `eval run --candidate X --tasks all`.

---

## 7. Kiến trúc & Layout repo đề xuất

```
agent-eval-playground/
├── tasks/                 # bộ task (xem §4)
│   ├── t1-.../
│   └── t2-.../
├── candidates/            # các prompt/skill/workflow đang đánh giá
│   ├── baseline/
│   └── github-pack-abc/
├── runner/                # harness
│   ├── run.py             # orchestrate: reset → run → grade
│   ├── sandbox.py         # tạo môi trường sạch (git worktree / docker)
│   ├── metrics.py         # đo token/tool/time
│   └── judge.py           # LLM-as-judge (blind, pairwise)
├── results/               # JSON kết quả từng run
│   └── 2026-05-29/...
├── leaderboard.md         # bảng so sánh (auto-generate)
└── eval.yaml              # config: model, trọng số, số lần lặp
```

### Sandbox/reset — 2 lựa chọn
- **Git worktree** (nhẹ, nhanh): mỗi run = 1 worktree mới từ commit pinned. Đủ cho task không cần cách ly hệ thống.
- **Docker container** (chắc chắn, đắt hơn): khi task đụng filesystem/network/system state. Khuyến nghị cho task-set "thật".

---

## 8. Đừng build lại từ 0 — Framework có sẵn

Trước khi tự viết harness, đánh giá fork:

| Framework | Điểm mạnh | Phù hợp khi |
|-----------|-----------|-------------|
| **inspect-ai** (UK AISI) | Framework eval tổng quát, có sandbox + scorer + LLM-judge sẵn | Muốn harness linh hoạt, ít boilerplate |
| **SWE-bench / SWE-bench-verified** | Chuẩn vàng cho "fix bug trong repo thật", có hidden test | Task-set T1/T3 kiểu thật |
| **Terminal-Bench** | Đánh giá agent thao tác terminal/CLI end-to-end | Task cần shell/CLI |

→ Khuyến nghị: **fork inspect-ai làm harness**, mượn format task kiểu SWE-bench. Chỉ tự viết phần leaderboard + candidate-loader đặc thù team.

---

## 9. Rủi ro & Cách giảm

| Rủi ro | Hệ quả | Giảm thiểu |
|--------|--------|-----------|
| State rò rỉ giữa các run | Số liệu vô nghĩa | Reset snapshot sạch tuyệt đối |
| Variance cao của LLM | Kết luận sai từ 1 lần chạy | Chạy ≥3 lần, báo cáo pass@k + std |
| Contamination (task nổi tiếng) | Agent "thuộc bài" | Dùng task tự chế / ít phổ biến |
| LLM-judge thiên vị | Xếp hạng sai | Blind + pairwise + human calibrate |
| Nhảy vào T4 quá sớm | Nhiễu, nản, tốn công | Vững T1–T2 trước |
| Cost benchmark cao | Tốn token | Giới hạn timeout/tool-call, cache, chạy song song có kiểm soát |

---

## 10. Lộ trình triển khai

**Phase 0 — Spike (1–2 ngày)**
- Dựng 2 task T1/T2 + 1 candidate baseline.
- Viết runner tối giản: reset (git worktree) → run → verify.sh → in pass/fail + token.

**Phase 1 — MVP (1 tuần)**
- 5–10 task T1/T2.
- Auto objective grade + đo cost. Leaderboard markdown.
- Chạy ≥3 lần/task, báo cáo pass@k.

**Phase 2 — Chất lượng (1–2 tuần)**
- Thêm LLM-judge (blind + pairwise) cho code quality.
- Thêm 3–5 task T3.
- Quy trình human calibrate.

**Phase 3 — Mở rộng (sau)**
- Task T4 (idea→app) với rubric + smoke test.
- Tích hợp CI: candidate mới push → tự chạy eval → comment leaderboard diff.

---

## 11. Định nghĩa "thành công"

- Thêm 1 candidate mới → chạy 1 lệnh → có bảng so sánh trong < 30 phút (không tính compute).
- Số liệu objective ổn định: cùng candidate chạy lại, pass@k dao động < 1 task.
- LLM-judge khớp human ≥ 80% trên mẫu calibrate.
- Team ra quyết định "dùng prompt-pack nào" dựa trên leaderboard, không còn cảm tính.
