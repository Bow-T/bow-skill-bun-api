---
name: bun-queue-worker
description: Thêm hoặc sửa job chạy qua hàng đợi (pgmq hoặc queue trong Postgres) và worker tiêu thụ nó — payload có schema và tương thích ngược, handler chạy lại không gây hại, hạn hiển thị đủ dài, chỉ xoá message sau khi đã ghi kết quả, chặn job độc lặp vô hạn, chuyển trạng thái có điều kiện, tắt worker êm và dọn tài nguyên. Dùng khi thêm loại job / queue mới, đổi payload, sửa vòng lặp worker, retry, timeout, số worker chạy song song; hoặc khi gặp "job chạy hai lần", "job kẹt", "job mất", "worker treo". Thêm queue hay loại job mới thì đi cùng [[ts-union-variant]]; chạy thử bằng [[bun-local-stack]].
---

# bun-queue-worker — Job có thể chạy hai lần; viết sao cho không sao

## Điều phải nhớ trước tiên

Hàng đợi kiểu "đọc rồi ẩn tạm" giao message **ít nhất một lần**. Worker chết, quá hạn hiển
thị, hay deploy giữa chừng đều làm message hiện lại và được worker khác nhận. Mọi quyết định
bên dưới xuất phát từ đó.

## Bước 1 — Đọc cách repo đang làm

Tìm và đọc trước khi sửa: nơi enqueue, vòng lặp đọc của worker, chỗ xoá / lưu trữ message,
nơi khai tên queue, schema payload, bảng trạng thái job, cách worker tắt. Job mới làm theo
đúng đường của một job anh em đã chạy ổn.

## Bước 2 — Payload

- Payload có schema ở package dùng chung, validate **cả khi enqueue lẫn khi worker đọc**.
- Thay đổi payload phải tương thích ngược: lúc deploy, queue còn message theo khuôn cũ, và
  worker bản cũ còn có thể đọc message khuôn mới. Thêm trường tuỳ chọn thì an toàn; đổi tên,
  bỏ trường, đổi nghĩa thì làm hai bước qua hai lần release.
- Payload chỉ mang id và tham số, không mang dữ liệu cá nhân hay secret: message nằm lại
  trong bảng queue và bảng lưu trữ.

## Bước 3 — Handler chạy lại được

- Kết quả ghi bằng upsert theo khoá của job, hoặc kiểm "đã xong chưa" trước khi làm.
- Tác dụng phụ ra ngoài (gửi mail, gọi API, ghi file) có khoá chống lặp, hoặc đặt sau bước
  ghi trạng thái có điều kiện.
- Chuyển trạng thái bằng câu lệnh có điều kiện, ví dụ `UPDATE … SET status = 'running' WHERE
  id = $1 AND status = 'queued'`, rồi kiểm số dòng bị đổi. Không dòng nào đổi nghĩa là worker
  khác đã nhận: bỏ qua, không làm tiếp.

## Bước 4 — Hạn hiển thị, xoá, và job độc

Với pgmq: `pgmq.read(queue, vt, qty)` ẩn message trong `vt` giây; `pgmq.delete` hoặc
`pgmq.archive` mới là xác nhận xong; mỗi lần đọc lại làm `read_ct` tăng.

- `vt` phải dài hơn thời gian xử lý lâu nhất có thể, hoặc worker gia hạn trong lúc chạy.
  `vt` ngắn hơn job là nguyên nhân số một của "job chạy hai lần".
- Xoá / lưu trữ message **sau khi** kết quả đã ghi xong. Xoá trước rồi worker chết là mất job.
- Đặt trần số lần thử theo `read_ct`. Vượt trần thì lưu trữ message, đánh dấu job thất bại
  kèm lý do, và dừng. Không có trần thì một message hỏng sẽ quay vòng mãi và chặn cả queue.
- Lỗi do input (payload sai schema, bản ghi không còn) là lỗi vĩnh viễn: thất bại ngay, đừng retry.

## Bước 5 — Enqueue

- Queue nằm cùng Postgres với dữ liệu thì enqueue **trong cùng transaction** với lần ghi tạo
  ra job. Không cùng transaction được thì enqueue sau khi commit, và có đường quét lại job
  đã tạo mà chưa được enqueue.
- Tên queue là hằng ở package dùng chung. Queue mới cần đủ ba thứ: migration tạo queue, chỗ
  định tuyến chọn queue, và worker đọc nó. Thiếu một là job nằm đó không ai xử lý, không lỗi nào hiện ra.

## Bước 6 — Vòng đời worker

- Nhận tín hiệu tắt thì ngừng đọc message mới, chờ job đang chạy xong trong hạn cho phép,
  quá hạn thì để message tự hiện lại. Không xoá message của job chưa xong.
- Mọi bước gọi ra ngoài (trình duyệt, thiết bị, tiến trình con, mạng) có timeout cứng.
- Tài nguyên mở trong job (trình duyệt, tiến trình con, file tạm, kết nối) được đóng trong
  `finally`. Worker sống lâu mà rò mỗi job một tiến trình con thì vài giờ sau máy hết tài nguyên.
- Số job chạy song song trên một worker có trần rõ ràng, khớp với tài nguyên của máy chạy nó.
- Log mỗi chặng kèm id job; không log payload nguyên văn nếu nó có thể mang dữ liệu người dùng.

## Bước 7 — Test

1. Chạy handler **hai lần** với cùng một message: kết quả và tác dụng phụ như chạy một lần.
2. Message theo khuôn payload cũ vẫn xử lý được (hoặc thất bại sạch, có lý do).
3. Handler ném lỗi giữa chừng: message không bị xoá, trạng thái không kẹt ở "running" vĩnh viễn.
4. Vượt trần số lần thử: message được lưu trữ, job mang trạng thái thất bại.

## Bước 8 — Verify runtime

Đẩy một job thật qua queue và theo id của nó tới kết quả ([[bun-local-stack]]). Với thay đổi
về retry hoặc tắt worker: tắt worker giữa lúc job chạy, bật lại, xác nhận job hoàn tất đúng
một lần về mặt kết quả.

## Báo cáo

- Queue và loại job đã đụng; payload đổi gì, có tương thích ngược không.
- `vt`, trần số lần thử, timeout đang đặt bao nhiêu và vì sao.
- Đã test chạy lặp chưa; đã chạy thật một job chưa, quan sát được gì.
