# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay phần giữ chỗ dưới mỗi câu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Hoàng Việt  Mã học viên: 2A202601940

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Ví dụ, khi deploy service công khai nhưng quên cấu hình `AGENT_API_KEY` trên
> Render, ứng dụng sẽ dừng ngay lúc khởi động và log báo thiếu biến môi trường.
> Nhờ vậy tôi phát hiện lỗi trước khi service nhận request. Nếu dùng khóa mặc
> định như `"changeme"`, ứng dụng vẫn chạy; người khác có thể đoán khóa, gọi API
> và làm phát sinh chi phí trước khi tôi nhận ra cấu hình bị thiếu.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log tôi nhận được có dạng:
>
> ```json
> {"event":"ask_completed","level":"info","timestamp":"2026-08-10T04:10:00+00:00","user_id":"sv-test","tokens_in":12,"tokens_out":24,"cost_usd":0.000048}
> ```
>
> Vì log là JSON có các trường riêng, tôi có thể lọc và đếm request theo
> `user_id`, đồng thời cộng `cost_usd` để theo dõi chi phí của từng người dùng.
> Tôi cũng có thể tạo cảnh báo theo `level` hoặc theo số lỗi trong một khoảng
> thời gian. Dòng `print("đã trả lời xong")` không chứa đủ dữ liệu có cấu trúc để
> thực hiện các việc đó.

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
| 1 stage (bản đầu) | Chưa đo được trên máy hiện tại |
| Multi-stage | Chưa đo được trên máy hiện tại |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Docker Desktop trên máy kiểm tra chưa chạy nên tôi chưa có số MB đáng tin cậy
> để ghi vào bảng; tôi không dùng số liệu ước đoán thay cho kết quả thật. Tuy
> nhiên, image multi-stage đã build thành công trên GitHub Actions. Về nguyên
> tắc, phần chênh lệch chủ yếu là công cụ build, cache và các tệp trung gian chỉ
> cần lúc cài dependency. Stage runtime chỉ nhận thư viện đã cài cùng source cần
> chạy nên không mang toàn bộ môi trường của builder sang production.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile hiện tại, các layer base image, `WORKDIR`, `COPY
> requirements.txt` và `RUN pip install` vẫn được lấy từ cache khi tôi chỉ sửa
> `app/main.py`. Layer `COPY app/ app/` và các layer đứng sau nó phải tạo lại.
> Nếu đặt `COPY . .` trước `RUN pip install`, chỉ một thay đổi nhỏ trong source
> cũng làm layer copy đổi, kéo theo việc cài lại toàn bộ dependency, khiến build
> chậm hơn dù `requirements.txt` không thay đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu ứng dụng Python có lỗ hổng thực thi lệnh, kẻ tấn công có thể chạy lệnh với
> quyền của process trong container. Khi process chạy bằng root, họ có quyền
> root bên trong container; nếu container còn được cấp mount hoặc capability
> nguy hiểm, họ có thể đọc/ghi tài nguyên của host và mở rộng phạm vi tấn công.
> Lệnh `USER appuser` làm process chỉ có UID 10001 không đặc quyền, nên mã bị
> khai thác không mặc nhiên có quyền root. Đây là lớp giảm thiểu thiệt hại, dù
> vẫn cần cấu hình mount và capability an toàn.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Người dùng có thể gửi tối đa 20 request trong hai giây nằm hai bên ranh giới
> phút: gửi 10 request ở giây 59 của phút trước và 10 request ở giây 00 của phút
> sau. Bộ đếm theo phút đã reset nên cả hai nhóm đều hợp lệ, dù thực tế có 20
> request trong khoảng hai giây. Sliding window 60 giây vẫn nhìn thấy nhóm đầu
> khi nhóm sau tới nên ngăn được burst này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn số request trong một cửa sổ thời gian, còn cost guard giới
> hạn tổng tiền đã tiêu trong tháng. Ví dụ, một user chỉ gửi 2 request/phút nhưng
> mỗi request rất dài và tốn nhiều token: rate limit cho qua, còn cost guard sẽ
> chặn khi vượt ngân sách. Ngược lại, một user gửi 11 câu rất ngắn trong một
> phút khi vẫn còn gần như toàn bộ ngân sách: cost guard vẫn cho phép về mặt chi
> phí, nhưng rate limit chặn request thứ 11 bằng HTTP 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu `/health` kiểm tra cả Redis, khi Redis mất kết nối thì cả ba container đều
> trả health check lỗi. Orchestrator xem cả ba instance là không khỏe và lần
> lượt restart chúng, dù process ứng dụng vẫn hoạt động bình thường. Các
> container mới khởi động vẫn không kết nối được Redis nên tiếp tục bị đánh dấu
> lỗi và restart, tạo thành vòng lặp, làm sự cố Redis lan thành mất toàn bộ
> service. Tách `/health` và `/ready` giúp process vẫn sống, còn load balancer
> chỉ tạm ngừng gửi traffic cho instance chưa sẵn sàng.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với lịch sử lưu trong Redis, các request của cùng `X-User-Id` dùng chung dữ
> liệu dù Nginx chuyển chúng qua những container khác nhau, nên
> `history_length` tăng nhất quán theo số lượt hội thoại (mỗi request ghi thêm
> một message user và một message assistant). Nếu dùng dict Python trong từng
> process, mỗi container có một bản lịch sử riêng; khi request bị round-robin,
> tôi sẽ thấy `history_length` lúc tăng, lúc quay về giá trị thấp tùy container
> nhận request. Dữ liệu đó cũng mất khi container restart.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi tôi gặp là workflow GitHub Actions có job `test` và `build` thành công
> nhưng job `deploy` thất bại ngay tại bước `Trigger Render deployment`; smoke
> test vì thế bị bỏ qua và badge báo failure. Tôi mở chi tiết workflow và xác
> định lệnh đang đọc `${{ secrets.RENDER_DEPLOY_HOOK_URL }}`, trong khi Deploy
> Hook chưa được cấu hình đúng trong GitHub Actions. Tôi sao chép Deploy Hook từ
> Render, tạo Repository secret đúng tên `RENDER_DEPLOY_HOOK_URL`, rồi chạy lại
> failed jobs. Sau đó cả ba job đều xanh, smoke test `/health` thành công và bộ
> test bonus CI/CD đạt 13/13.
