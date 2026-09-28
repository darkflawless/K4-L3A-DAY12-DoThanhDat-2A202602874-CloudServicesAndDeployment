# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay thế các phần trả lời bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đỗ Thành Đạt  Mã học viên: 2A202602874

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống: Khi deploy service lên cloud (như Render hay Railway), nếu tôi quên cấu hình biến môi trường `AGENT_API_KEY`.
- Nếu để mặc định `"changeme"`: Service vẫn khởi động thành công (trả 200 OK) và public ra internet. Bất kỳ ai trên mạng đều có thể dùng key mặc định `"changeme"` để gọi API `/ask`, gửi hàng loạt câu hỏi và tiêu cạn toàn bộ ngân sách/tài khoản LLM của tôi.
- Khi áp dụng Fail-Fast (chết sớm): Thiếu secret thì Pydantic ném `ValidationError` và container sập ngay lập tức (Exit code 1). Dashboard trên cloud sẽ báo đỏ `Deploy Failed`, buộc tôi phải nhận ra lỗi cấu hình và bổ sung API key an toàn trước khi public service ra ngoài.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được từ service:
```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T08:31:00.123456+00:00", "user_id": "sv-test", "tokens_in": 12, "tokens_out": 35, "cost_usd": 0.00047}
```

Hai việc làm được với dòng log JSON mà `print("đã trả lời xong")` không làm được:
1. **Lọc và truy vấn chính xác theo từng trường (Structured Querying):** Các hệ thống gom log tập trung trên Cloud (như Datadog, Grafana Loki, CloudWatch, Render) tự động parse JSON thành các trường. Ta có thể viết truy vấn: `user_id == "sv-test" AND cost_usd > 0.01` để trích xuất chính xác hành vi của user. Lệnh `print` thường chỉ sinh chuỗi text không thể bóc tách số liệu tự động.
2. **Tổng hợp chỉ số và bắn cảnh báo tự động (Metrics & Alerting):** Có thể tính tổng chi phí `cost_usd` và lượng token theo thời gian thực để vẽ đồ thị giám sát, hoặc tạo cảnh báo tự động (Alert) qua Slack/Telegram khi phát hiện tần suất lỗi hoặc chi phí tăng vọt.

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
| 1 stage (bản đầu) | ~1.02 GB |
| Multi-stage | ~214 MB |

Giải thích: Phần dung lượng chênh lệch (~800 MB) là toàn bộ các công cụ phục vụ quá trình biên dịch (như GCC, build-essential, Linux C headers), bộ nhớ cache tải về của pip, và các file tạm thời khi cài đặt thư viện. Stage builder dùng chúng để build wheels vào `/install`, sau đó stage runtime chỉ copy các package đã cài đặt sang image mới mà không mang theo compiler thừa thãi, giúp image siêu nhẹ và an toàn.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại: Khi chỉ sửa 1 ký tự trong `app/main.py`, các layer từ `FROM`, `WORKDIR`, `COPY requirements.txt .`, đến `RUN pip install ...` đều được dùng lại 100% từ Cache (vì `requirements.txt` không đổi). Chỉ có layer `COPY . .` và các lệnh phía sau nó mới phải chạy lại. Quá trình build diễn ra chỉ trong 1-2 giây.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi lần sửa code, layer `COPY . .` bị thay đổi sẽ làm vô hiệu hóa toàn bộ cache của các layer phía sau. Docker sẽ bị ép chạy lại lệnh `RUN pip install` từ đầu, phải tải và cài lại toàn bộ thư viện, làm thời gian build kéo dài nhiều phút một cách lãng phí.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện:
1. Code Python có lỗ hổng (như Remote Code Execution qua deserialization, exec/eval, hoặc upload file ghi đè).
2. Kẻ tấn công khai thác lỗ hổng để thực thi mã độc. Do container chạy mặc định bằng root (UID 0), mã độc sẽ có đặc quyền root bên trong container.
3. Kẻ tấn công khai thác thêm lỗ hổng container escape (như lỗ hổng kernel Linux, cgroup, hoặc mount volume nhạy cảm) để thoát ra ngoài container. Vì UID bên trong là 0, kẻ tấn công sẽ có quyền **root trên chính hệ điều hành máy host**, kiểm soát toàn bộ server.
- Lệnh `USER appuser` cắt đứt chuỗi ở bước 2: Tiến trình Python chỉ chạy với quyền của một người dùng thông thường bị cô lập. Dù app bị chiếm quyền, kẻ tấn công cũng không thể sửa đổi file hệ thống hay leo thang kiểm soát máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa: **20 requests** trong 2 giây liên tiếp.
Giải thích:
Với cơ chế Fixed Window (reset lúc giây 00):
- Ở giây 59 của phút trước (10:00:59): Người dùng gửi 10 requests -> Hợp lệ (10/10).
- Ở giây 00 của phút sau (10:01:00): Đồng hồ bước sang phút mới, counter được reset về 0. Người dùng lập tức gửi tiếp 10 requests -> Vẫn hợp lệ (10/10).
Tổng cộng trong 2 giây (từ 10:00:59 đến 10:01:00), hệ thống phải hứng chịu 20 requests liên tục, gấp đôi hạn mức 10 req/phút. Cửa sổ trượt (Sliding Window 60s) giải quyết triệt để vấn đề này vì nó luôn tính tổng request trong 60 giây gần nhất tính từ thời điểm hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Khác nhau:
- **Rate Limit:** Giới hạn theo **số lượng request** trong một đơn vị thời gian (tần suất gọi API).
- **Cost Guard:** Giới hạn theo **số tiền chi phí thực tế (USD)** tích lũy trong một chu kỳ (bảo vệ ngân sách tài chính).

Hai tình huống:
1. *Rate limit cho qua nhưng Cost guard chặn (402):* User cả ngày mới gửi 1 request duy nhất (Rate limit: 1/10 req/phút -> cho qua). Nhưng request đó nhồi prompt dài kèm ngữ cảnh khổng lồ tốn $12 tiền token LLM, trong khi hạn mức ngân sách tháng của user chỉ còn $1 -> Cost guard chặn lại với mã lỗi `402 Payment Required`.
2. *Cost guard cho qua nhưng Rate limit chặn (429):* User gửi liên tục 40 request trong 5 giây. Mỗi câu hỏi cực ngắn chỉ tốn $0.0001 (tổng mới tốn $0.004, còn rất xa hạn mức $10/tháng) -> Cost guard cho phép, nhưng Rate limit sẽ chặn từ request thứ 11 với mã lỗi `429 Too Many Requests` để bảo vệ server khỏi nguy cơ nghẽn mạng và quá tải (DDoS).

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện xảy ra:
1. Mạng chập chờn hoặc Redis restart khiến kết nối Redis bị gián đoạn trong 30 giây.
2. Bộ điều phối (Docker Compose / Kubernetes / Cloud Orchestrator) gửi liveness probe định kỳ vào endpoint `/health`.
3. Do endpoint gọi kiểm tra Redis và thất bại, `/health` trả về mã lỗi 503 hoặc timeout.
4. Liveness probe thất bại khiến Orchestrator nhận định rằng toàn bộ tiến trình ứng dụng đã bị chết/treo, lập tức gửi tín hiệu KILL và RESTART toàn bộ 3 container.
5. Cả 3 container bị restart liên tục thành vòng lặp (CrashLoopBackOff) trong suốt 30 giây Redis chết. Toàn bộ hệ thống bị sập hoàn toàn và không thể phục vụ bất kỳ ai.
*(Nếu tách riêng: `/health` trả 200 giúp container không bị restart; `/ready` trả 503 giúp load balancer tạm thời không đẩy traffic vào cho đến khi Redis kết nối lại bình thường).*

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu lịch sử lưu trong dict Python của từng container:
Do 3 container agent là 3 tiến trình riêng biệt với vùng nhớ RAM hoàn toàn độc lập, khi Load Balancer phân phối các request kế tiếp:
- Request 1 vào container A: A lưu vào dict của mình -> response trả về `history_length = 0`.
- Request 2 rơi vào container B: Dict của B rỗng -> response trả về `history_length = 0` (agent quên sạch câu trước).
- Request 3 rơi vào container C: Dict của C cũng rỗng -> response trả về `history_length = 0`.
- Request 4 quay lại container A: A tìm thấy câu 1 trong RAM của nó -> `history_length = 2`.
Kết quả: `history_length` sẽ nhảy lung tung (0, 0, 0, 2, 2, 4...) tùy theo request rơi trúng container nào, khiến hội thoại bị đứt gãy. Khi lưu ở Redis tập trung, mọi container cùng thấy một trạng thái duy nhất và `history_length` sẽ tăng đều đặn (0 -> 2 -> 4 -> 6...).

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Lỗi:** Quên cấu hình biến môi trường `AGENT_API_KEY` trong tab Environment trên Cloud Platform (như Render).
- **Thông báo lỗi:** Container bị crash lúc khởi động với log: `pydantic_core._pydantic_core.ValidationError: 1 validation error for Settings agent_api_key Field required`.
- **Cách tìm nguyên nhân:** Mở tab **Logs** của service trên Render Dashboard, thấy log đỏ báo lỗi Pydantic Settings do thiếu trường bắt buộc `agent_api_key`.
- **Cách khắc phục:** Vào mục **Environment** trên Dashboard, thêm biến `AGENT_API_KEY` với một chuỗi khóa bí mật an toàn, sau đó nhấn **Save Changes** để Render tự động redeploy lại service thành công.
