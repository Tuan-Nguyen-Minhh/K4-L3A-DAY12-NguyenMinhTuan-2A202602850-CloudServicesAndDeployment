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

> Ví dụ deploy lên production nhưng quên cấu hình AGENT_API_KEY. Nếu để "changeme", app vẫn chạy và có thể gửi request bằng key mặc định. Với fail fast, app dừng ngay khi khởi động, giúp phát hiện lỗi cấu hình trước khi đưa service vào sử dụng.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Ví dụ log:{"event":"ask_completed","level":"info","timestamp":"2026-09-28T...","cost_usd":0.002} Có thể lọc/thống kê chi phí qua cost_usd,có thể tìm và phân tích sự kiện theo event, thời gian hoặc level,trong khi print("đã trả lời xong") chỉ cho biết có một thông báo, không có dữ liệu có cấu trúc để máy phân tích.

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
| 1 stage (bản gốc `HEAD:Dockerfile`, `FROM python:3.11`) | build không thành công — retry 2 lần 28/09 đều fail y hệt ở `RUN pip install -r requirements.txt`: PyPI timeout 5 lần ở `/simple/watchfiles/` rồi pip backtrack hết 40 bản `uvicorn[standard]` → `ResolutionImpossible`, `exit code: 1`, nên không có image final để đo |
| Multi-stage (`Dockerfile` hiện tại, `python:3.11-slim` 2 stage) | `day12-agent:multi 272MB` (build fresh), `day12-agent:cp2-test 272MB`, `day12-agent:prod 354MB` (bản cũ, context thừa); base `python:3.11-slim` chỉ `189MB` |

> Giải thích: chênh lệch nằm ở (1) base full `python:3.11` (~1GB qua các layer pull thấy ở log) so với `slim` chỉ `189MB`; (2) stage `builder` giữ pip cache/wheel/compiler rồi bị vứt đi, stage runtime chỉ `COPY --from=builder /install` + dùng `pip install --no-cache-dir`; (3) `.dockerignore` loại `.venv/.git/__pycache__/.env` nên `COPY . .` không mang rác vào image. Cả các bản multi đo được đều < 500MB nên pass `test_image_du_nho`.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> `Dockerfile` hiện tại `COPY requirements.txt` + `RUN pip install` trước `COPY . .` (`Dockerfile:28-35`). Sửa 1 ký tự trong `app/main.py` chỉ invalidate từ layer `COPY . .` trở đi; các layer `FROM`, `COPY requirements.txt`, `RUN pip install`, `COPY --from=builder` được reuse từ cache, build lại chỉ mất vài giây. Nếu đặt `COPY . .` lên trước `RUN pip install` như bản gốc `HEAD:Dockerfile`, mọi sửa code đều làm cache miss từ sớm nên phải chạy lại `pip install` (layer nặng nhất, vài phút) dù thư viện không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Lỗ hổng RCE trong Python (ví dụ `eval`/`pickle.loads` với input user) cho attacker shell trong container. Nếu process là root (uid 0, mặc định khi thiếu `USER`), attacker kết hợp escape qua `docker.sock` bị mount nhầm, volume host, `--privileged` hoặc kernel exploit là thành root host: đọc/ghi file host, sang container khác. `Dockerfile:36-37` tạo user `tuan` và `USER tuan` nên dù RCE vẫn chỉ là uid thường — không mount socket, không ghi `/etc`, không load kernel module, blast radius dừng trong container.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa 20 request trong 2 giây. Cách đạt: gửi 10 request lúc 10:00:59 (hết quota phút 10:00) rồi ngay 10 request lúc 10:01:01 (quota phút 10:01 reset). Mỗi phút đồng hồ vẫn "đúng luật" 10/phút nhưng thực tế 20 request dồn trong 2 giây. Sliding window 60 giây của `app/rate_limiter.py:20` đếm 60 giây gần nhất nên không có kẽ hở này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit (`app/rate_limiter.py`) giới hạn *số lượng* request/60s, cost guard (`app/cost_guard.py`) giới hạn *số tiền* user/tháng. Rate cho qua nhưng cost chặn: 10 request/phút đều dưới limit nhưng mỗi request kèm lịch sử dài 50k token → `spent + estimated > budget` → 402. Ngược lại: user mới tháng mới (`spent=0`) gửi 15 request rẻ trong 1 phút → cost còn xa budget nhưng request thứ 11+ bị 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện: (1) Redis mất kết nối 30s; (2) cả 3 container đều trả 503 ở endpoint gộp vì nó kiểm tra Redis; (3) orchestrator hiểu 503 = unhealthy nên restart cả 3 cùng lúc, load balancer cũng rút cả 3 khỏi vòng xoay; (4) Redis quay lại nhưng không còn container nào phục vụ (tất cả đang restarting/cold start) → user thấy 502. Tách riêng thì `/health` (không chạm Redis) vẫn 200 nên container không bị restart, chỉ `/ready` 503 để LB tạm ngừng gửi traffic — sự cố nhỏ không thành outage.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis (`app/store.py`) `history_length` tăng đơn điệu 0, 2, 4... vì mọi container cùng đọc một list. Với dict trong RAM, mỗi container có bộ nhớ riêng nên con số nhảy lung tung theo container phục vụ request đó: 0, 0, 2, 0... — agent "mất trí nhớ" ngẫu nhiên, đúng lý do `tests/test_cp4.py` bắt buộc state nằm ngoài process.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi build bản 1-stage (`agent:single`, `FROM python:3.11`): `RUN pip install -r requirements.txt` fail `ResolutionImpossible` sau khi PyPI timeout 5 lần ở `/simple/watchfiles/`. Tìm ra bằng cách đọc log build (thấy base pull OK, chỉ fail ở step pip) rồi test riêng: tải trang đó từ máy host mất 24.6s/824KB (quá timeout 15s của pip 24.0), trong container còn chậm hơn. Cách xử lý: bỏ bản gốc, deploy bản multi-stage lên Railway, set đủ biến rồi Generate Domain — verify cả 4 endpoint xanh ngay lần đầu.
