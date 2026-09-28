# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đoàn Anh Quân -  Mã học viên: 2A202602803

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Việc "chết sớm" sẽ làm cho app sập ngay khi khởi động, chặn quá trình deploy và báo lỗi ngay cho tôi, từ đó tôi biết lỗi ở đâu. Còn để mặc định "changeme" thì app vẫn sống, nhưng gọi /ask là bằng tiền của bạn, sau này chỉ có thể phát hiện khi nhìn hoá đơn, lúc đó đã tốn rất nhiều tiền rồi.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T07:42:47.798099+00:00", "user_id": "sv-cp4-demo", "tokens_in": 43, "tokens_out": 44, "cost_usd": 3.285e-05}\
Từ dòng log này tôi có thể lọc theo `user_id` để biết được ai đang hỏi gì, hỏi bao nhiêu, ... Ngoài ra còn có thể cộng `cost_usd` để biết tổng số tiền đã tiêu, biết mỗi người tiêu bao nhiêu tiền.

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
| 1 stage (bản đầu) | 1730 MB |
| Multi-stage | 297 MB |

Phần chênh lệch dung lượng đến từ :
- base image `python3.11` với đầy đủ gcc, header, thư viện ,...
- COPY cả `venv/`, `tests/`, ...
- cache của pip


---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Sửa 1 ký tự thì `COPY requirements.txt, pip install, useradd, COPY --from=builder` dùng lại được cache, layer `COPY app` và `COPY utils` phải chạy lại.
- Đặt `COPY ..` trước `RUN pip install` thì Docker huỷ cache từ layer đầu tiên thay đổi trở đi. `COPY..` thay đổi, kéo theo `pip install` chạy lại. Layer `COPY from=buider` cũng mất cache.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện: Lỗ hổng code Python -> Quyền `root` trong container -> Thoát container -> Chiếm quyền máy host.\
Lệnh User cắt đứt chuỗi đó ở phần quyền, buộc app chạy bằng user thường. Kẻ tấn công nếu hack thì chỉ ở quyền hạn thấp, không đủ quyền để thao túng hay thoát khỏi host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa 20 request. Người đó gửi 10 request vào giây 59 đợt trước (giây cuối trước khi reset) và 10 request vào giây 1 đợt sau (giây đầu sau khi reset). 

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- Rate limit thì đếm số lần gọi còn cost guard đếm tổng số tiền.
- Tình huống 1: Mỗi phút được 5 request, những mỗi câu hỏi thì dài 2000 ký tự, cost guard chặn lại vì quá tốn sau vài ngày.
- Tình huống 2: 50 câu "hello" trong 1 phút, chi phí rẻ nhưng vượt 10 request/phút nên chặn lại.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Gộp 2 endpoint và kiểm tra Redis:
Redis mất kết nối  -> 3 container báo unhealthy -> Reset cả 3 container cùng lúc -> Không còn container nào nên lỗi 5xx -> Redis quay lại sau 30 giây nhưng container vẫn đang khời động lại, có thể fail vì chưa nối được Redis.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- Gọi 5 lần cùng 1 user thì `history_length` là 0, 2, 4, 6, 8, các request chia ra: agent-1: 2 lần, agent-2: 2 lần và agent-3: 1 lần.
- Nếu lịch sử lưu trong 1 dict Python thay vì Redis, mỗi container chỉ nhớ phần của nó, nên các số sẽ nhảy lung tung. Rơi vào container nào mới thì sẽ là 0, container reset cũng là 0.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

`railway.toml` có `startCommand = "uvicorn ... --port $PORT"`. Railway chạy lệnh này không qua shell nên `$PORT` không được thay giá trị, và uvicorn sẽ báo lỗi cổng không hợp lệ. Lỗi được phát hiện khi đọc lại cấu hình trước khi deploy. Cách sửa: bỏ `startCommand` để dùng `CMD ["sh", "-c", "exec uvicorn ... --port ${PORT:-8000}"]` trong Dockerfile.
