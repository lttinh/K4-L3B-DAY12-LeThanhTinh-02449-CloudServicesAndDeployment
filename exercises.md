# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Họ và tên: Lê Thanh Tịnh  
> Mã học viên: 02449

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Theo em, fail fast có ích nhất khi deploy app lên cloud. Ví dụ nếu em quên chưa cấu hình `AGENT_API_KEY`, app sẽ lỗi ngay lúc khởi động. Nhờ vậy em biết luôn nguyên nhân là thiếu biến môi trường và sửa ngay được.
>
> Nếu để mặc định là `"changeme"` thì app vẫn có thể chạy, nhưng tới lúc gọi API thật mới phát sinh lỗi. Khi đó nhìn bên ngoài service vẫn chạy bình thường nên sẽ khó tìm lỗi hơn. Vì vậy để app dừng sớm sẽ giúp phát hiện sai cấu hình nhanh hơn và tránh việc dùng nhầm key giả trong môi trường thật.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Hiện tại em mới thấy access log của `/health`, chưa lấy được dòng JSON `ask_completed` nên em chưa điền một dòng log giả.
>
> Khi có log JSON dạng:
>
> ```json
> {"event":"ask_completed", ...}
> ```
>
> thì em có thể làm được ít nhất hai việc mà `print("đã trả lời xong")` không làm được.
>
> Thứ nhất, em có thể lọc và thống kê theo từng trường trong log, ví dụ theo `user_id`, trạng thái request, thời gian phản hồi hoặc số lần gọi API.
>
> Thứ hai, log JSON có thể đưa vào hệ thống monitoring để tạo dashboard hoặc cảnh báo khi có lỗi, latency cao hoặc request bất thường.
>
> Còn `print("đã trả lời xong")` chỉ là một câu text chung chung, không có cấu trúc nên rất khó dùng để truy vấn hoặc thống kê tự động.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|---|---:|
| 1 stage (bản đầu) | Chưa đo |
| Multi-stage | 272 MB |

> Kết quả em đo được với image multi-stage là **272 MB**, nhỏ hơn mức yêu cầu 500 MB.
>
> Hiện tại em chưa build và đo lại bản 1 stage nên em để là chưa đo thay vì tự điền số liệu.
>
> Theo em, phần dung lượng chênh lệch giữa 1 stage và multi-stage chủ yếu là các thứ chỉ cần lúc build như compiler, build tools, cache, file tạm và một số dependency trung gian. Với multi-stage thì các phần này nằm ở build stage và không được mang sang image cuối cùng. Vì vậy image cuối chỉ giữ những gì cần để chạy app nên nhẹ hơn.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi em chỉ sửa một ký tự trong `app/main.py`, các layer ở phía trước phần copy source vẫn có thể lấy lại từ cache, ví dụ base image, `WORKDIR`, copy file dependency và `RUN pip install`, miễn là file dependency không đổi.
>
> Từ layer copy source code trở đi thì Docker phải chạy lại vì nội dung source đã thay đổi.
>
> Nếu đặt `COPY . .` lên trước `RUN pip install` thì chỉ cần em sửa một file Python cũng làm layer `COPY` thay đổi. Khi đó các layer phía sau cũng bị mất cache, nên `pip install` phải chạy lại dù `requirements.txt` không đổi.
>
> Vì vậy nên copy file dependency trước, cài package trước, rồi mới copy source code để tận dụng cache tốt hơn.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Ví dụ app Python có một lỗ hổng cho phép người khác thực thi lệnh từ xa. Nếu container đang chạy bằng root thì sau khi khai thác lỗ hổng, người tấn công cũng có quyền root bên trong container.
>
> Nếu container lại có thêm cấu hình không an toàn hoặc có lỗ hổng container escape thì từ quyền root trong container, họ có thể tiếp tục truy cập tài nguyên của máy host hoặc leo thang đặc quyền.
>
> Khi dùng lệnh `USER` để chạy app bằng một user thường, dù lỗ hổng Python vẫn bị khai thác thì người tấn công chỉ nhận được quyền của user đó chứ không phải root. Như vậy `USER` không sửa trực tiếp lỗ hổng, nhưng giúp giảm quyền mà người tấn công có thể lấy được sau khi khai thác.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Người dùng có thể gửi tối đa **20 request trong khoảng 2 giây**.
>
> Ví dụ ở giây 59 của phút hiện tại, họ gửi đủ 10 request. Sang giây 00 của phút tiếp theo, bộ đếm được reset nên họ gửi tiếp 10 request nữa.
>
> Như vậy trong khoảng rất ngắn quanh lúc chuyển phút có thể có 20 request, dù giới hạn ghi là 10 request/phút.
>
> Sliding window sẽ tránh được trường hợp này vì nó luôn xét 60 giây gần nhất, không phụ thuộc vào mốc đổi phút.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Theo em, rate limit giới hạn số lượng hoặc tần suất request trong một khoảng thời gian, còn cost guard giới hạn chi phí hoặc tài nguyên mà request tiêu tốn.
>
> Ví dụ rate limit vẫn cho qua nếu người dùng mới gửi 2 request trong giới hạn 10 request/phút. Nhưng nếu 2 request đó rất dài, dùng nhiều token hoặc làm chi phí gần vượt ngân sách thì cost guard phải chặn request tiếp theo.
>
> Trường hợp ngược lại, người dùng có thể gửi nhiều request rất ngắn nên tổng chi phí vẫn thấp, cost guard chưa chặn. Nhưng nếu số request vượt 10 request trong 60 giây thì rate limit vẫn phải chặn.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Đầu tiên Redis bị mất kết nối.
>
> Sau đó endpoint health của cả 3 container cùng kiểm tra Redis và đều trả về trạng thái lỗi.
>
> Hệ thống orchestration có thể coi cả 3 container là unhealthy rồi restart hoặc loại chúng ra khỏi service.
>
> Nếu Redis vẫn chưa lên thì các container mới khởi động lại vẫn tiếp tục fail health check và có thể lặp lại việc restart.
>
> Trong khi đó bản thân process Python có thể vẫn đang sống, chỉ là không kết nối được Redis.
>
> Nếu tách `/health` và `/ready` thì `/health` chỉ kiểm tra process còn sống hay không, nên container không bị restart vô ích. `/ready` mới kiểm tra Redis. Khi Redis mất kết nối, container chỉ tạm thời không nhận traffic. Khi Redis hoạt động lại thì `/ready` thành công và service nhận request trở lại.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Khi em chạy:
>
> ```bash
> docker compose up -d --scale agent=3
> ```
>
> thì việc scale lên 3 instance bị thất bại.
>
> Lỗi em nhận được là:
>
> ```text
> Bind for 0.0.0.0:8000 failed: port is already allocated
> ```
>
> Nguyên nhân là trong file Compose đang map cố định:
>
> ```text
> 8000:8000
> ```
>
> nên chỉ một container có thể bind vào host port 8000. Instance thứ hai và thứ ba không thể dùng lại cùng host port đó.
>
> Vì vậy ở lần chạy này em chưa lấy được chuỗi `history_length` thực tế của 3 instance.
>
> Muốn scale được thì cần bỏ việc map cố định cùng một host port cho từng agent và đặt reverse proxy hoặc load balancer ở phía trước.
>
> Về mặt state, nếu dùng Redis thì các instance cùng đọc và ghi vào một vùng dữ liệu chung nên cùng một `X-User-Id` sẽ có lịch sử thống nhất.
>
> Nếu thay bằng dict Python thì mỗi container có một dict riêng. Khi request bị phân phối sang các container khác nhau, `history_length` sẽ tăng riêng ở từng instance, nên có thể thấy dạng như `1, 1, 2, 1, 3...` chứ không tăng liên tục. Nếu container restart thì phần history nằm trong dict của container đó cũng mất.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Khi deploy bằng Render Blueprint, em gặp vấn đề ở phần biến môi trường `AGENT_API_KEY`.
>
> Trên màn hình deploy, Render yêu cầu nhập giá trị cho `AGENT_API_KEY` nhưng lúc đầu em chưa điền. Trong code thì `agent_api_key` là biến bắt buộc và không có giá trị mặc định, nên nếu thiếu biến này app không thể khởi động đúng.
>
> Em tìm ra nguyên nhân bằng cách xem lại phần cấu hình Blueprint trên Render và đối chiếu với phần `Settings` trong code.
>
> Sau đó em nhập API key hợp lệ vào phần Environment của Render rồi deploy lại.
>
> Em không ghi trực tiếp API key vào source code hoặc đẩy lên GitHub mà để dưới dạng biến môi trường để tránh lộ key.
