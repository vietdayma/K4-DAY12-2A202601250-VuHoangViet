# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng gợi ý bên dưới mỗi câu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Vũ Hoàng Việt  Mã học viên: 2A202601250

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Lần đầu deploy lên Railway tôi quên set `API_TOKEN` trong dashboard. Vì
> field này không có default, app crash ngay lúc khởi động, Railway báo
> lỗi Deployment failed luôn nên tôi biết ngay để vào set lại, chưa kịp có
> ai gọi vào service cả.
>
> Nếu để mặc định `changeme` thì chắc app vẫn khởi động bình thường,
> `/healthz` với `/readyz` đều xanh, nhìn tưởng ổn. Nhưng vì `changeme`
> nằm sẵn trong code mà repo này lại public, ai đọc được code cũng gọi
> `/chat` thành công bằng đúng chuỗi đó. Lúc đó chắc chỉ phát hiện ra khi
> xem hóa đơn LLM tăng bất thường vài ngày sau, chứ không phải ngay lúc
> deploy. Nên cách chết sớm này đúng là cứu tôi thật — thà lỗi hiện ngay
> lúc build còn hơn để lộ lỗ hổng âm thầm cả tuần.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Gọi `/chat` xong tôi lấy được dòng log này:
>
> ```
> {"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T09:49:13.111357+00:00", "client_id": "sv-test", "prompt_tokens": 3, "completion_tokens": 37, "usd_cost": 2.265e-05}
> ```
>
> Hai việc tôi làm được mà `print()` không làm được: thứ nhất là lọc/gom
> theo field — trên dashboard log của Railway tôi có thể tìm kiểu
> `event="chat_completed" AND usd_cost > 0.001`, hoặc gom theo `client_id`
> để xem client nào tốn tiền nhất, còn log dạng chữ tự do thì phải viết
> regex đoán mò và dễ vỡ mỗi khi tôi đổi câu chữ. Thứ hai là nhờ `ts` theo
> chuẩn ISO-8601 và `severity` viết hoa, hệ thống log hiểu được và có thể
> vẽ biểu đồ `usd_cost` theo thời gian, thậm chí tự bắn cảnh báo khi chi
> tiêu vượt ngưỡng trong 1 giờ — không cần ngồi đọc log bằng mắt.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t chat:single .
docker build -t chat:multi .
docker images | grep chat
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | 67.8 MB (67 764 465 bytes) |
| Multi-stage | 63.7 MB (63 690 115 bytes) |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Chênh lệch tôi đo được chỉ khoảng 4MB (~6%), ít hơn tôi nghĩ lúc đầu.
> Lý do là cả hai bản đều dùng chung base `python:3.11-slim` và đều
> `pip install --no-cache-dir`, nên không có chuyện tích cache wheel to
> đùng. Cái mà bản multi-stage cắt được là ở chỗ stage cuối chỉ
> `COPY --from=builder /install /usr/local` — tức chỉ lấy đúng thư mục cài
> đặt cuối cùng, không mang theo `requirements.txt`, không có thư mục
> `/build`, không dính bất cứ file trung gian nào khác của quá trình build.
> Nếu app cần cài package có C-extension mà không có sẵn wheel cho bản
> `slim` (phải cài thêm `build-essential`/`gcc` để biên dịch) thì chênh
> lệch chắc chắn lớn hơn nhiều, vì toolchain đó chỉ tồn tại ở stage
> `builder` và không bao giờ lọt vào image chạy thật.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Tôi thêm một dòng comment vào `app/main.py` rồi build lại, xem log thấy:
> `WORKDIR /build`, `COPY requirements.txt .`, `RUN pip install ...` (cả
> stage `builder`) và `COPY --from=builder /install /usr/local` đều được
> cache lại (log ghi `CACHED`), vì `requirements.txt` không đổi. Còn
> `COPY app/ app/`, `COPY utils/ utils/` và cả `RUN useradd ...` thì chạy
> lại — dù `useradd` chẳng liên quan gì tới file trong `app/`, nó vẫn
> build lại vì nằm sau layer bị đổi trong Dockerfile. Docker cache theo
> tuyến tính: một layer cache-miss thì mọi layer phía sau nó cũng
> cache-miss theo, bất kể có phụ thuộc thật hay không.
>
> Nếu đặt `COPY . .` lên trước `RUN pip install` thì mỗi lần tôi sửa dù
> chỉ một dòng code, `COPY . .` sẽ cache-miss, kéo theo `RUN pip install`
> cũng cache-miss dù `requirements.txt` chẳng đổi gì — tức là mỗi lần build
> đều phải tải và cài lại toàn bộ dependency, chậm đi rất nhiều so với hiện
> tại trong vòng lặp code → build → test hằng ngày.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện tôi hình dung: có một lỗ hổng trong code Python (ví dụ
> một thư viện parse input không an toàn) bị khai thác, khiến attacker
> chạy được lệnh shell bên trong container. Vì container mặc định chạy
> bằng UID 0 (root), lệnh đó có toàn quyền trên filesystem và tiến trình
> bên trong container. Nếu container còn bị cấu hình sai ở đâu đó (mount
> Docker socket, chạy `--privileged`, hoặc lỗi kernel cho phép escape
> namespace), thì root-trong-container trở thành bàn đạp để leo thang
> thành root-trên-host.
>
> `USER appuser` (không tạo home dir, shell `/usr/sbin/nologin`) cắt đứt
> chuỗi này ngay ở bước có UID root: dù attacker vẫn chạy được lệnh, tiến
> trình giờ chạy dưới một UID thường, không ghi được vào phần lớn
> filesystem, không cài được package hệ thống, không sửa được binary — mọi
> con đường leo thang tiếp theo cần quyền root làm bàn đạp đều bị chặn lại
> ngay tại đây.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> `WWW-Authenticate` là bắt buộc theo chuẩn HTTP cho mọi response 401 — nó
> nói cho client biết chính xác phải xác thực bằng cơ chế nào (ở đây là
> `Bearer`), để client tự biết cần gắn `Authorization: Bearer <token>` ở
> lần gọi sau thay vì đoán mò hay thử `Basic Auth`.
>
> Còn việc trả cùng một thông báo cho cả ba trường hợp là vì nếu phân biệt
> rõ giữa thiếu header, sai scheme và sai token, một kẻ đang dò token bằng
> cách thử hàng loạt giá trị sẽ biết được lúc nào họ đoán gần đúng so với
> lúc sai hoàn toàn — thông tin đó thu hẹp không gian brute-force của họ.
> Trả về đúng một câu `invalid or missing bearer token` cho mọi trường
> hợp thì việc dò không có tín hiệu phản hồi nào để bám vào.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> `refill_per_second = 10/60 = 1/6` token/giây. Xô chỉ cần 60 giây im lặng
> là đầy lại 100% rồi, nên im lặng tới 10 phút (600 giây) cũng không làm
> nó đầy hơn, vì `min(capacity, tokens)` chặn cứng ở 10. Client gửi liên
> tiếp gần như không có thời gian trôi giữa các request nên không refill
> thêm gì trong lúc gửi — kết quả là gửi được đúng 10 request thành công,
> request thứ 11 mới bị 429.
>
> Nếu bỏ `min(capacity, ...)` đi thì sau 600 giây im lặng, `tokens` sẽ
> bằng trạng thái cũ cộng thêm `600 × 1/6 = 100`. Nếu trước đó xô gần cạn
> (ví dụ 0 token), sau 10 phút im lặng nó sẽ tích được tới 100 token thay
> vì bị chặn ở 10 — client gửi liên tiếp được tận 100 request trước khi
> bị 429, gấp 10 lần capacity thiết kế ban đầu. Nói cách khác xô không còn
> giới hạn trên nữa, client chỉ cần im lặng đủ lâu là vượt qua được rate
> limit.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> Với hạn mức $30/tháng, `guard.check` chỉ chặn khi tổng chi cả tháng vượt
> $30 — nó không quan tâm client chi nhanh trong một đêm hay chi rải rác
> cả tháng. Nếu sự cố xảy ra đầu tháng, client có thể đốt hết $30 chỉ
> trong một đêm, thiệt hại tối đa là $30, và vì ngân sách chỉ reset đầu
> tháng sau nên service bị khóa (402) cho client đó gần cả tháng mới tự
> hồi phục.
>
> Với hạn mức $1/ngày như trong lab, khóa Redis là
> `spend:{client_id}:{ngày UTC}`, nên thiệt hại tối đa của một sự cố chỉ
> là $1. Vì `CostGuard.today()` dùng ngày UTC, khóa tự động đổi sang ngày
> mới lúc 0h UTC — dù sự cố bắt đầu lúc 2h sáng, tới nửa đêm cùng ngày
> (giờ UTC) là service đã có ngân sách mới, tự hồi phục trong tối đa 24
> giờ mà không cần ai can thiệp thủ công. Đúng như lý do ghi trong
> docstring của `cost_guard.py`: hạn mức ngày giới hạn thiệt hại xuống
> 1/30 so với hạn mức tháng.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp `/healthz` và `/readyz` thành một endpoint có check Redis, với
> cụm 3 container khi Redis mất kết nối 30 giây thì tôi nghĩ chuyện sẽ
> diễn ra thế này: cả 3 container cùng gọi Redis lúc probe định kỳ và đều
> lỗi kết nối, nên endpoint gộp trả 503 cho cả liveness lẫn readiness cùng
> lúc, vì giờ chỉ còn một tín hiệu cho hai câu hỏi khác nhau. Vì endpoint
> này giờ cũng là liveness probe, orchestrator hiểu 503 liên tục nghĩa là
> process đã chết chứ không phải chỉ sống nhưng chưa sẵn sàng, nên sau vài
> lần fail liên tiếp nó sẽ restart cả 3 container gần như cùng lúc (vì
> Redis mất kết nối ảnh hưởng cả cụm chứ không phải lỗi riêng một
> instance). Trong lúc cả 3 container đang restart, cụm có 0 instance sẵn
> sàng nhận traffic, nên toàn bộ request thật của người dùng bị lỗi
> 502/503 suốt khoảng thời gian đó — dù vấn đề gốc chỉ là Redis chớp mắt
> 30 giây rồi tự phục hồi. Container khởi động lại và thử kết nối Redis
> lần nữa, nếu lúc đó Redis đã sống lại thì pass, nhưng cụm vừa trải qua
> một đợt outage hoàn toàn không cần thiết.
>
> Còn với cách tách riêng như code hiện tại thì `/healthz` không đụng
> Redis nên luôn 200 suốt 30 giây đó, không container nào bị restart; chỉ
> `/readyz` trả 503 nên load balancer chỉ ngừng đẩy traffic mới vào (request
> đang xử lý dở vẫn xong bình thường), cụm tự rút khỏi rotation rồi tự vào
> lại khi Redis sống lại — không có container nào bị giết oan.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi tôi gặp là `/readyz` trả 503 (`{"status":"not ready","redis":false}`)
> liên tục dù `/healthz` vẫn 200 bình thường — service vẫn sống nhưng
> chưa bao giờ sẵn sàng. Nguyên nhân là `REDIS_URL` tôi gõ tay lúc đầu trên
> Railway không khớp với địa chỉ thật của Redis add-on. Tôi tìm ra bằng
> cách xem log container trên dashboard Railway, thấy lỗi kết nối Redis
> ngay sau dòng log `service_started`, rồi gọi `curl <URL>/readyz` để xác
> nhận đúng là 503 kèm `"redis": false` — từ đó khoanh vùng được là do kết
> nối Redis chứ không phải lỗi trong code app. Cách sửa là đổi `REDIS_URL`
> sang dùng Variable Reference của Railway (biến tự trỏ theo đúng kết nối
> do Redis add-on cung cấp) thay vì gõ tay, để sau này nếu add-on đổi
> host/port thì biến vẫn tự cập nhật đúng mà không cần sửa lại thủ công.
