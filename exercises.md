# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> Câu trả lời của bạn` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lục Minh Đức  Mã học viên: 2A202601918

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Lúc deploy lên Render mà quên set biến AGENT_API_KEY, app crash ngay với lỗi ValidationError, nhìn log thấy liền và biết phải thêm biến. Nếu để mặc định "changeme" thì app vẫn chạy bình thường, ai cũng gọi được /ask bằng khóa "changeme" mà không ai biết, mỗi request tốn tiền LLM. Chết sớm giúp phát hiện lỗi ngay lúc deploy thay vì khi đã mất tiền.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Gọi /ask vài lần thấy log dạng: `{"event":"ask_completed","level":"info","timestamp":"2026-08-10T11:53:56+00:00","user_id":"sv-test","tokens_in":90,"tokens_out":47,"cost_usd":4.17e-05}`. Với dòng log này có thể: (1) lọc ra tất cả request có cost_usd cao bất thường để tìm user nào tiêu nhiều tiền nhất, (2) đếm số event có level "error" trong 5 phút qua để tính tỷ lệ lỗi rồi cảnh báo tự động. Còn print("đã trả lời xong") thì không lọc, không tổng hợp, không cảnh báo gì được.

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
| 1 stage (bản đầu) | ~1.0 GB |
| Multi-stage | ~200 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần chênh lệch khoảng 800MB chủ yếu là build-essential (gcc, g++, make...) và các file header dùng để biên dịch thư viện Python có phần C. Trong bản 1 stage, tất cả đều nằm trong image cuối. Bản multi-stage chỉ copy kết quả đã cài xong từ stage builder sang stage runtime, compiler và file trung gian bị bỏ lại nên image nhẹ hơn nhiều.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile hiện tại, thứ tự là COPY requirements.txt trước, rồi pip install, rồi mới COPY app. Khi sửa 1 ký tự trong main.py thì layer requirements.txt không đổi nên pip install được dùng lại từ cache, chỉ layer COPY app phải chạy lại — build rất nhanh, vài giây. Nếu đặt COPY . . lên trước pip install thì bất kỳ thay đổi nào trong code cũng làm cache của pip install bị huỷ, phải cài lại toàn bộ thư viện mỗi lần build, mất vài phút.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Giả sử code Python có lỗ hổng command injection, kẻ tấn công gửi input độc hại qua /ask và thực thi được lệnh shell bên trong container. Nếu container chạy bằng root thì lệnh đó chạy với quyền root, kẻ tấn công có thể đọc ghi mọi file trong container, và nếu có thêm lỗ hổng container escape thì leo lên được quyền root trên máy host. Lệnh USER appuser cắt đứt chuỗi ở bước thực thi lệnh: dù kẻ tấn công chạy được lệnh trong container, lệnh đó chỉ có quyền của user thường, không đọc được file hệ thống, không cài thêm package, và không thể exploit tiếp để leo quyền lên host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa 20 request trong 2 giây. Cách làm: gửi 10 request vào lúc 10:00:59 (vẫn nằm trong phút 10:00, chưa vượt hạn mức), rồi gửi thêm 10 request lúc 10:01:01 (sang phút mới, bộ đếm reset về 0 nên lại được 10 lần nữa). Kết quả là 20 request trong 2 giây mà vẫn "đúng luật". Sliding window không có kẽ hở này vì nó luôn nhìn lại 60 giây gần nhất, 10 request ở giây 59 vẫn bị đếm khi gọi tiếp ở giây 01.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit kiểm soát tốc độ gọi (bao nhiêu request/phút), cost guard kiểm soát tổng chi phí (bao nhiêu USD/tháng). Tình huống rate limit cho qua nhưng cost guard chặn: user gọi 5 request/phút (dưới hạn mức 10), nhưng mỗi request dùng 50.000 token rất tốn tiền, sau vài ngày tổng chi phí vượt 10 USD thì cost guard chặn dù tốc độ gọi vẫn thấp. Tình huống ngược lại: đầu tháng mới, user chưa tốn đồng nào (cost guard cho qua) nhưng gửi 15 request trong 1 phút thì rate limit chặn ở request thứ 11.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> (1) Redis mất kết nối. (2) Endpoint gộp kiểm tra Redis, thấy không kết nối được, trả 503 cho cả 3 container. (3) Orchestrator thấy cả 3 container báo unhealthy, bắt đầu restart lần lượt hoặc cùng lúc cả 3. (4) Trong lúc restart, không còn container nào sống để phục vụ request, user thấy lỗi 502. (5) Redis quay lại sau 30 giây nhưng cả 3 container vẫn đang restart, chưa sẵn sàng. (6) Sự cố nhỏ (Redis mất kết nối 30 giây) biến thành sự cố toàn hệ thống (downtime vài phút). Nếu tách riêng thì /health không kiểm tra Redis nên không trigger restart, chỉ /ready trả 503 để load balancer ngừng gửi request tạm thời, khi Redis quay lại thì mọi thứ tự phục hồi.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis, history_length tăng đều: 0, 2, 4, 6, 8... vì cả 3 container cùng đọc ghi trên một Redis chung, dù request rơi vào container nào thì lịch sử vẫn liên tục. Nếu dùng dict Python thì mỗi container có dict riêng trong RAM, history_length sẽ nhảy lung tung kiểu 0, 0, 2, 0, 2, 4... vì request bị load balancer chia vào container khác nhau, mỗi container chỉ biết những request nó từng xử lý, agent "mất trí nhớ" ngẫu nhiên.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Deploy lần đầu lên Render thì service start rồi tắt ngay, log báo ConnectionError: Error 111 connecting to localhost:6379. Nhìn vào thì hiểu ra: trong container trên cloud, localhost là chính container đó chứ không phải máy local, nên không có Redis ở đó. Vào dashboard Render kiểm tra thì biến REDIS_URL chưa được gắn vào service. Sau khi tạo Redis add-on trên Render và để nó tự gắn biến REDIS_URL vào service, deploy lại thì /ready trả 200 và mọi thứ hoạt động bình thường.
