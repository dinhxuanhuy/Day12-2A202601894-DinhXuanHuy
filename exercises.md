# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay placeholder bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đinh Xuân Huy  Mã học viên: 2A202601894

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu để mặc định `agent_api_key = "changeme"`, khi deploy ứng dụng lên server mà người quản trị quên không khai báo biến `AGENT_API_KEY`, app vẫn khởi động bình thường. Kẻ xấu có thể quét và dùng key mặc định `"changeme"` để gọi `/ask`, vừa làm rò rỉ dữ liệu vừa tiêu tốn hàng ngàn USD chi phí LLM. Với cơ chế Fail-fast (không có mặc định), app crash ngay lập tức khi khởi động, buộc lập trình viên phải bổ sung key hợp lệ trước khi service khởi chạy, ngăn ngừa triệt để sự cố lộ lỗ hổng bảo mật ra sản phẩm thật.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
```json
{"timestamp": "2026-08-10T11:11:41.123456+00:00", "event": "ask_completed", "user_id": "sv-test", "tokens_in": 12, "tokens_out": 25, "cost_usd": 0.00015}
```

Hai việc làm được với log JSON:
1. **Truy truy vấn và phân tích tự động bằng hệ thống Central Logging (Elasticsearch/Loki/Datadog):** Hệ thống có thể tự động parse JSON để lọc chính xác theo `user_id`, tính tổng chi phí `cost_usd` hay tạo biểu đồ thống kê mức độ sử dụng theo thời gian mà không cần viết regex phức tạp.
2. **Tạo quy tắc cảnh báo tự động (Automated Alerting):** Có thể trích xuất các thông số định lượng để tự động kích hoạt cảnh báo (ví dụ: gửi mail/slack khi `cost_usd > 0.05` ở 1 request hoặc tổng token vọt cao bất thường), giúp phát hiện sự cố/tấn công tức thì.

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
| 1 stage (bản đầu) | 320 MB |
| Multi-stage | 270 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~50 MB) là bộ nhớ cache của pip, các tập tin thư viện tạm thời và công cụ biên dịch intermediate phát sinh trong quá trình `pip install` ở builder stage. Multi-stage build chỉ copy kết quả thư mục thư viện đã cài hoàn chỉnh sang image cuối, giúp loại bỏ toàn bộ rác build không cần thiết khi chạy ứng dụng trên môi trường production.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Khi sửa 1 ký tự trong `app/main.py`:
- Các layer `FROM`, `WORKDIR`, `COPY requirements.txt .`, `RUN pip install...` được sử dụng lại từ cache vì `requirements.txt` không thay đổi.
- Layer `COPY app/ app/` và các layer phía sau bị hỏng cache và bắt buộc phải chạy lại.

Nếu đặt `COPY . .` lên **trước** `RUN pip install`: mỗi khi thay đổi bất kỳ file mã nguồn nào (`app/main.py`), layer `COPY . .` sẽ bị invalid cache, làm cho lệnh `RUN pip install` ở ngay sau **phải tải và cài đặt lại toàn bộ thư viện từ đầu**, khiến thời gian build image bị kéo dài rất nhiều.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện:
1. Xuất hiện lỗ hổng thực thi mã từ xa (RCE) trong code Python (ví dụ: qua `eval()`, `pickle`, hoặc command injection).
2. Kẻ tấn công gửi payload độc hại để thực thi lệnh hệ thống bên trong container dưới quyền `root` (UID 0).
3. Do container chạy dưới quyền `root`, kẻ tấn công lợi dụng các lỗ hổng container escape (hoặc mount socket/volume từ host) để truy cập và can thiệp file hệ thống máy chủ host với quyền root.

Lệnh `USER appuser` cắt đứt chuỗi sự kiện tại **bước 2**: tiến trình Python chạy dưới tài khoản unprivileged (`appuser`). Dù kẻ tấn công khai thác được lỗi RCE trong ứng dụng Python, tiến trình mã độc vẫn bị hạn chế tối đa quyền hạn, không thể đọc/ghi các tài nguyên nhạy cảm và bị chặn đứng khi cố thực hiện container escape.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.
**Giải thích:** Người dùng gửi 10 request ở giây `10:00:59` (nằm trong phút 10:00) và ngay sau đó gửi tiếp 10 request ở giây `10:01:01` (nằm trong phút 10:01). Do bộ đếm reset về 0 ở giây `10:01:00`, cả 2 lần gửi đều được hệ thống chấp nhận là hợp lệ ở 2 phút khác nhau, nhưng thực tế người dùng đã thực hiện 20 request dồn dập chỉ trong khoảng thời gian 2 giây.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- **Khác biệt:** Rate Limit quản lý **tần suất/số lượng request** trong cửa sổ thời gian ngắn (ví dụ: 10 req/phút). Cost Guard quản lý **tổng ngân sách chi tiêu tài chính (USD/token)** tích lũy trong khoảng thời gian dài (ví dụ: $10/tháng).
- **Rate Limit cho qua nhưng Cost Guard chặn:** User gửi 1 request duy nhất trong phút (đúng luật Rate Limit), nhưng request này xử lý đoạn văn bản cực lớn làm chi phí vượt quá số tiền còn lại trong tháng $\rightarrow$ Cost Guard phát hiện hết ngân sách và chặn (402).
- **Cost Guard cho qua nhưng Rate Limit chặn:** User còn rất nhiều ngân sách tháng ($100), nhưng liên tục gửi 15 request ngắn chỉ trong vòng 3 giây $\rightarrow$ Rate Limit chặn lại do spam dồn dập (429).

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

1. Khi Redis mất kết nối, endpoint gộp trả về mã lỗi `503`.
2. Orchestrator (Docker/Kubernetes) lầm tưởng tiến trình ứng dụng Python bị treo hoặc chết (Health check thất bại).
3. Orchestrator lập tức **tiêu diệt (kill) và khởi động lại (restart) toàn bộ 3 container** ứng dụng.
4. Do Redis vẫn chưa khôi phục, các container vừa khởi động lại tiếp tục health check thất bại và tiếp tục bị restart liên tục (CrashLoopBackOff), gây nghẽn tài nguyên máy chủ và kéo dài thời gian sập của cả hệ thống.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Con số `history_length` sẽ **thay đổi nhấp nhô, không tăng đều (ví dụ: 1, 1, 1, 2, 2...)**.
**Nguyên nhân:** Nginx Gateway chia tải 5 request liên tiếp tới 3 container khác nhau theo cơ chế Round-Robin. Do mỗi container dùng 1 dict Python riêng trên RAM local, thông tin lưu ở Container A không xuất hiện ở Container B/C, dẫn đến mỗi container đếm lịch sử độc lập và làm agent bị "mất trí nhớ" tùy thuộc vào container nhận request.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Lỗi gặp phải:** App không bind đúng cổng của môi trường cloud làm Health check bị timeout (`Health check failed on port 8000: connection refused`).
- **Cách tìm nguyên nhân:** Đọc log deployment trên giao diện điều khiển của Cloud Platform, thấy server Uvicorn mặc định lắng nghe cố định ở cổng `8000`, trong khi Cloud Platform tự động gán một cổng ngẫu nhiên qua biến môi trường `$PORT`.
- **Cách sửa:** Cập nhật câu lệnh khởi chạy Uvicorn trong Dockerfile thành `CMD ["sh", "-c", "exec uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]` để ứng dụng luôn ưu tiên lấy cổng từ biến môi trường `$PORT` của Cloud Platform.
