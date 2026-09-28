# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay mỗi dòng trích dẫn mẫu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyen Minh Tuan  Mã học viên: 2A202602850

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Quên set `AGENT_API_KEY` trên Railway: không default nên pydantic ném `ValidationError` lúc boot, thấy ngay trong log. Để `"changeme"` thì app vẫn chạy, attacker dùng key mặc định gọi chùa `/ask` đốt tiền LLM mà chỉ phát hiện khi nhìn hóa đơn.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Log: `{"event":"ask_completed","level":"info","timestamp":"2026-09-28T...","cost_usd":0.002}`. Làm được: (1) `sum(cost_usd)` theo user tìm ai tiêu nhiều nhất; (2) tỉ lệ `level=error`/5 phút để alert. `print()` plain không lọc hay aggregate được.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng (`docker images` ngày 28/09) |
|-----|------------------------------|
| 1 stage (bản gốc, `FROM python:3.11`) | build fail, không có image — PyPI timeout ở `/simple/watchfiles/` rồi pip backtrack → `ResolutionImpossible` |
| Multi-stage (`python:3.11-slim`) | `day12-agent:multi 272MB`; base `slim` chỉ `189MB` |

> Chênh lệch là: base full ~1GB so với `slim`; stage `builder` bị vứt đi, runtime chỉ giữ kết quả (`--no-cache-dir`, `COPY --from=builder`); `.dockerignore` loại `.venv/.git/__pycache__/.env`.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Sửa `app/main.py` chỉ invalidate từ `COPY . .` trở đi; các layer `pip install` được reuse. Đặt `COPY . .` trước `RUN pip install` thì mỗi sửa code đều cài lại toàn bộ thư viện.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> RCE trong Python cho attacker shell trong container; nếu chạy root, escape qua `docker.sock`/volume/`--privileged`/kernel exploit là thành root host. `USER tuan` giữ attacker ở uid thường nên chuỗi leo quyền dừng trong container.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa 20 request: 10 lúc 10:00:59 + 10 lúc 10:01:01, mỗi phút vẫn "đúng luật". Sliding window đếm 60 giây gần nhất nên không có kẽ hở này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit chặn theo *số request*/60s, cost guard chặn theo *tiền*/tháng. Ít request nhưng mỗi request nặng token → qua rate, dính 402. Nhiều request rẻ → chưa hết budget nhưng dính 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Redis chết 30s → cả 3 container 503 → orchestrator restart cả 3, LB rút cả 3 → Redis về nhưng không còn ai phục vụ → 502. Tách riêng: `/health` 200 giữ container sống, `/ready` 503 chỉ rút traffic.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Redis: `history_length` tăng đều 0, 2, 4... Dict trong RAM: nhảy lung tung theo container (0, 0, 2, 0...) — agent "mất trí nhớ" ngẫu nhiên.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Build bản 1-stage fail `ResolutionImpossible`: PyPI timeout ở `/simple/watchfiles/` (host tải mất 24.6s, quá timeout 15s của pip). Fix: bỏ bản gốc, deploy bản multi-stage lên Railway + đủ biến → 4 endpoint xanh ngay.
