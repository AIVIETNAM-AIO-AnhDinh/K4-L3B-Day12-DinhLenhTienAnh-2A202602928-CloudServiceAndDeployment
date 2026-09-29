# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Dinh Lenh Tien Anh  Mã học viên: 2A202602928

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống mình gặp thật khi deploy lên Railway: lần deploy đầu, service `agent` chưa có biến `AGENT_API_KEY`. Lúc đó app chỉ đọc `Settings` ở request đầu tiên nên container vẫn khởi động, `/health` trả 200 và Railway báo deploy **SUCCESS** — nhưng mọi `/ask` đều trả 500. Mình đã thêm `get_settings()` vào `lifespan` để đọc cấu hình ngay lúc khởi động; chạy thử không có biến thì app dừng ngay với `agent_api_key Field required ... Application startup failed`, nên platform sẽ báo deploy FAILED và giữ bản cũ đang chạy.

Nếu để mặc định `"changeme"` thì tình huống trên còn tệ hơn: app chạy bình thường với khóa `changeme`, và vì giá trị mặc định nằm trong code của một repo công khai, bất kỳ ai đọc repo cũng gọi được `/ask` bằng `X-API-Key: changeme` — tức là dùng LLM miễn phí bằng tiền của mình. Không có mặc định thì quên set secret = không deploy được, chứ không phải = mở cửa cho người lạ.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Hai dòng log thật khi gọi `/ask` 2 lần vào stack `docker compose` ở máy:

```
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T03:36:57.442915+00:00", "user_id": "sv-q2", "tokens_in": 3, "tokens_out": 37, "cost_usd": 2.265e-05}
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T03:36:57.450803+00:00", "user_id": "sv-q2", "tokens_in": 46, "tokens_out": 50, "cost_usd": 3.69e-05}
```

1. **Lọc và cộng theo trường:** vì mỗi dòng là JSON, công cụ log (Railway, Datadog, Loki...) tách được `user_id`, `cost_usd`, `tokens_in` thành trường riêng. Mình có thể lọc toàn bộ request của `sv-q2`, hoặc cộng `cost_usd` theo từng user trong ngày để biết ai tốn tiền nhất. Trên Railway mình thấy log đã được tách sẵn thành dạng `event="service_started" service="day12-agent" ...`. Với `print("đã trả lời xong")` thì chỉ có một chuỗi chữ, không biết của user nào, tốn bao nhiêu.
2. **Đặt cảnh báo và ghép log nhiều container:** có thể tạo alert khi số dòng `level="error"` tăng hoặc khi `cost_usd` của một user vượt ngưỡng. `timestamp` theo ISO-8601 UTC nên khi chạy 3 container, log của cả 3 xếp đúng thứ tự thời gian.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | 1.73 GB (≈ 1730 MB) |
| Multi-stage | 335 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần chênh lệch khoảng 1,4 GB chủ yếu đến từ base image: `python:3.11` nặng 1,62 GB còn `python:3.11-slim` chỉ 215 MB (đo bằng `docker images`). Bản đầy đủ mang theo những thứ chỉ cần lúc *build*, không cần lúc *chạy*: mình kiểm tra trong image 1 stage thấy có trình biên dịch `gcc 14.2.0`, thư mục header `/usr/include` 57 MB, cùng nhiều thư viện dev của Debian. Ngoài ra bản 1 stage còn giữ cache của pip ở `/root/.cache/pip` (17 MB) vì `pip install` không có `--no-cache-dir`.

Bản multi-stage chỉ gồm `python:3.11-slim` + thư mục `/opt/venv` đã cài xong (92 MB) + code `app/`, `utils/`; stage `builder` bị bỏ lại sau khi build. Image vẫn có thể nhỏ hơn nữa nếu tách thư viện test (`pytest`, `httpx`, `PyYAML`) ra khỏi `requirements.txt`, vì hiện chúng vẫn được cài vào image chạy thật.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Mình sửa `SERVICE_VERSION = "1.0.0"` thành `"1.0.1"` trong `app/main.py` rồi build lại: mất **2 giây**. Các bước từ `ENV`, `RUN python -m venv`, `COPY requirements.txt`, `RUN pip install`, `RUN groupadd/useradd` đến `COPY --from=builder /opt/venv` đều in `Using cache`. Chỉ từ `COPY app/ ./app/` trở xuống là chạy lại (`COPY app/`, `COPY utils/`, `USER`, `EXPOSE`, `HEALTHCHECK`, `CMD` — các bước sau chỉ là metadata nên gần như tức thì). Lý do: Docker so checksum từng layer, layer nào thay đổi thì nó và mọi layer phía sau phải build lại, còn phía trước thì dùng cache.

Mình cũng thử một Dockerfile đặt `COPY . .` **trước** `RUN pip install`: sửa đúng một ký tự như trên thì build lại mất **21 giây**, vì `COPY . .` đổi checksum nên `pip install` phải chạy lại — tải và cài lại toàn bộ 30 package. Trên cloud (Railway build lại mỗi lần deploy) thì mỗi lần sửa một dòng code đều phải cài lại thư viện.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện khi container chạy bằng root:

1. Code Python có lỗ hổng (ví dụ command injection, deserialize dữ liệu không tin cậy, hoặc một thư viện dính CVE) → kẻ tấn công chạy được lệnh với quyền của process.
2. Process là **root trong container** → đọc/ghi mọi file: sửa code để cài backdoor, `apt install` thêm công cụ, đọc toàn bộ file hệ thống.
3. Root trong container vẫn là uid 0 đối với kernel của host (khi không bật user namespace). Chỉ cần thêm một cấu hình sai (mount `/var/run/docker.sock`, mount thư mục host, chạy `--privileged`) hoặc một lỗ hổng runtime (như CVE-2019-5736 của runc — cần root trong container để ghi đè binary runc) là thoát ra được và thành **root trên host**.

Lệnh `USER app` cắt chuỗi ở bước 2: process chạy bằng user thường. Mình kiểm tra trong image của mình: `whoami` → `app (uid 999)`, thử `touch` vào thư mục code thì bị từ chối (code thuộc root), và image không có `gcc`. Kẻ tấn công không sửa được code, không cài thêm gì, các kiểu thoát container cần root bị chặn, và nếu có thoát ra thì cũng chỉ là uid 999 không có quyền trên host. Tuy vậy `USER` không bảo vệ được biến môi trường như `AGENT_API_KEY` — process vẫn đọc được — nên vẫn phải vá lỗ hổng gốc.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Tối đa **20 request** (gấp đôi hạn mức). Cách làm: gửi 10 request vào cuối phút, ví dụ 10:00:59, bộ đếm của phút 10:00 lên 10/10. Đến 10:01:00 bộ đếm reset về 0 nên gửi ngay 10 request nữa. 20 request đó chỉ cách nhau khoảng 1–2 giây nhưng mỗi phút đồng hồ vẫn "đúng luật" 10 request.

Mình mô phỏng đúng kịch bản này: 20 request rải trong 1,81 giây từ 10:00:59.00 đến 10:01:00.81. Cách đếm theo phút đồng hồ cho qua cả 20; còn `RateLimiter` sliding window trong `app/rate_limiter.py` chỉ cho qua 10, vì lúc 10:01:00 nó vẫn đếm 10 request của 60 giây gần nhất (các request lúc 10:00:59 vẫn nằm trong cửa sổ).

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn **số lượng request mỗi phút** (chống spam, bảo vệ server), còn cost guard giới hạn **số tiền mỗi tháng** của từng user (bảo vệ hóa đơn LLM). Một request có thể rẻ hoặc rất đắt tùy số token, nên đếm request không thay được đếm tiền.

- **Rate limit cho qua, cost guard chặn:** một user chỉ gửi 3–4 request/phút (dưới hạn mức 10) nhưng mỗi câu hỏi dài 2000 ký tự cộng lịch sử 20 tin nhắn; với LLM thật mỗi request có thể tốn vài cent đến vài chục cent. Sau vài ngày tổng chi vượt 10 USD → `/ask` trả **402** dù user chưa bao giờ bị 429. Trong test mình đặt chi tiêu tháng của user = 999 USD thì request đầu tiên đã bị 402.
- **Cost guard cho qua, rate limit chặn:** một script gửi liên tục những câu ngắn như `"test"`. Khi kiểm tra bản deploy trên Railway, mình gọi 15 lần liên tiếp: 9 lần 200 rồi 6 lần **429**, trong khi tổng chi phí mới khoảng 0,0002 USD — rất xa ngân sách 10 USD.

Cả hai đều được kiểm tra *trước* khi gọi LLM, vì tiền mất ở bước gọi LLM.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Nếu một endpoint vừa làm liveness vừa kiểm tra Redis, khi Redis mất kết nối 30 giây:

1. t = 0: Redis mất. Probe của **cả 3** container cùng fail, vì chúng dùng chung một Redis.
2. Sau vài lần probe fail liên tiếp (ví dụ 3 lần × 10 giây), orchestrator kết luận cả 3 container "chết" và **restart cả 3 gần như cùng lúc**. Request đang xử lý dở bị cắt.
3. Trong lúc restart, load balancer không còn instance nào → mọi request đều lỗi 502, kể cả những thứ không cần Redis.
4. Container khởi động lại xong nhưng nếu Redis vẫn chưa về thì probe lại fail → restart tiếp, và thời gian chờ giữa các lần restart tăng dần.
5. t = 30 giây: Redis quay lại, nhưng các container có thể đang giữa lần restart hoặc đang chờ → thời gian ngừng dịch vụ **dài hơn 30 giây**. Restart không sửa được Redis mà chỉ làm sự cố nặng thêm.

Khi tách ra: `/ready` trả 503 → load balancer ngừng gửi traffic, còn `/health` vẫn 200 → không ai restart container. Redis về thì `/ready` trở lại 200 và traffic chạy lại ngay. Mình đã thử trên stack local: dừng Redis thì `/ready` trả `{"status":"not ready","redis":false}` 503 trong 0,13 giây, `/health` vẫn 200; bật Redis lại thì `/ready` về 200 mà container `agent` không phải restart.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Vì service `agent` map cố định cổng `8000:8000` nên 3 container không thể cùng chiếm cổng 8000 của máy. Mình chạy 3 bản `agent` sau Nginx (dùng `nginx/nginx.conf` có sẵn) rồi gọi `/ask` 6 lần với cùng `X-User-Id: sv-q9`. Theo log từng container, Nginx chia lần lượt: agent-3, agent-1, agent-2, agent-3, agent-1, agent-2. Vậy mà `history_length` vẫn tăng đều: **0, 2, 4, 6, 8, 10**, vì cả 3 container đọc/ghi chung một Redis.

Nếu lịch sử nằm trong một dict Python thì mỗi container có dict riêng trong RAM của nó. Với đúng thứ tự trên, con số sẽ là **0, 0, 0, 2, 2, 2**: mỗi container chỉ nhớ những lượt rơi vào chính nó, nên agent "mất trí nhớ" 2/3 số lần. Khi deploy lại hoặc container bị restart, dict mất sạch và mọi thứ quay về 0.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

**Lỗi:** sau khi deploy lên Railway, `/health` trả 200 nhưng `/ready` trả `503 {"status":"not ready","redis":false}` và mọi `/ask` có API key đều 500. Log của service `agent` (`railway logs --service agent`) báo `redis.exceptions.TimeoutError: Timeout connecting to server`.

**Cách tìm nguyên nhân:**

1. Kiểm tra `REDIS_URL` của `agent` → trỏ đúng `redis.railway.internal:6379`, nên vấn đề nằm ở phía Redis.
2. Xem log service Redis (`railway logs --service Redis`) → lặp lại liên tục `/bin/sh: 1: exec: docker-entrypoint.sh: not found`: Redis chưa từng khởi động được, dù dashboard vẫn báo Online vì Redis không có health check.
3. Xem build log của deployment Redis → thấy các bước `COPY app/`, `RUN pip install -r requirements.txt` của chính Dockerfile của mình. Hóa ra lúc đầu thư mục đang được link với service Redis (`railway status` hiện `Linked service: Redis`), nên lần chạy `railway up` đã deploy code app **đè lên** service Redis. Start command của Redis không tìm thấy script của image Redis nên chết liên tục.

**Cách sửa:** tạo service `agent` riêng và `railway service link agent` để các lần `railway up` sau đi đúng chỗ; chạy `railway redeploy --service Redis --from-source` để deploy lại đúng image `redis:8.2`. Log Redis hiện `Ready to accept connections`, `/ready` trả 200, cả 5 lệnh kiểm tra trong `DEPLOYMENT.md` đều đúng và `pytest tests/test_cp5.py` pass 9/9. Bài học: luôn xem `railway status` trước khi `railway up`, và trạng thái "Online" trên dashboard chưa có nghĩa là process thật sự chạy được.
