# Thông Tin Deploy — Checkpoint 5 (phương án dự phòng LOCAL_FALLBACK)

> Service chạy bằng `docker compose` ở máy local vì chưa đăng ký được
> tài khoản Railway/Render. `pytest tests/test_cp5.py` tự chuyển sang kiểm tra
> `http://localhost:8000` khi `LOCAL_FALLBACK=true`.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Nguyen Minh Tuan |
| Mã học viên | 2A202602850 |
| Repo | https://github.com/Tuan-Nguyen-Minhh/K4-L3A-DAY12-NguyenMinhTuan-2A202602850-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | http://localhost:8000 (chạy local, chưa có URL công khai) |
| Platform | Local Docker Compose (đã cân nhắc Railway / Render, chưa đăng ký được tài khoản nên dùng phương án dự phòng) |
| Ngày deploy | 2026-09-28 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | compose map 8000:8000, cloud sẽ tự gán nên app đọc từ biến môi trường PORT |
| `AGENT_API_KEY` | ✅ | đặt trong file .env ở máy local, không nằm trong repo |
| `REDIS_URL` | ✅ | trỏ tới service redis trong compose, dạng redis://redis:6379/0 |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Chạy trên máy local với stack `docker compose up -d`:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i http://localhost:8000/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i http://localhost:8000/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: <lay-tu-file-env-may-ban>" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'
```

## Kết Quả Chạy Thật

Ngày 2026-09-28, stack `agent` + `redis` đều `healthy`:

```
{"status":"ok","service":"day12-agent","version":"1.0.0"}
{"status":"ready","redis":true}
POST /ask không key -> 401
POST /ask có key   -> 200 kèm answer, history_length, cost_usd
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/compose-ps.png` — terminal chạy `docker compose ps` và kết quả gọi API
- `screenshots/health.png` — kết quả gọi `/health` (xem trong ảnh compose-ps)

---

## Nếu Dùng Phương Án Dự Phòng

Không đăng ký được tài khoản cloud (Railway/Render đòi xác minh vượt quá thời gian
buổi lab), nên nộp theo phương án dự phòng, CP5 tối đa 60% điểm:

1. Đặt `LOCAL_FALLBACK=true` trong `.env` — đã làm
2. Chạy `docker compose up -d` rồi kiểm tra `docker compose ps` — cả 2 service `healthy`
3. Chụp màn hình vào `screenshots/` — đã có `compose-ps.png`
4. Chạy `pytest tests/test_cp5.py -v` — bộ test tự chuyển sang kiểm tra
   `http://localhost:8000`
5. Lý do không deploy được: chưa có tài khoản cloud (không có thẻ, đăng ký
   thất bại trong buổi lab) + mạng tới PyPI từ Docker rất chậm (build bản
   1-stage fail, xem câu 10 exercises.md), nên giữ bản multi-stage chạy local.
