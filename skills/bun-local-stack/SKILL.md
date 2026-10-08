---
name: bun-local-stack
description: Dựng và chạy thử một thay đổi trên stack NHIỀU TIẾN TRÌNH (DB + API + web + worker hàng đợi + thiết bị/emulator) ngay trên máy — tìm script khởi động và runbook của chính repo, bật lát cắt nhỏ nhất đủ chứng minh, đẩy MỘT job đi hết đường (UI/API → queue → worker → kết quả → UI), rồi báo cái gì đã chạy thật và cái gì chưa. Dùng khi cần "chạy thử", "mở app bấm thử", "test trên emulator/simulator", verify thay đổi UI hoặc worker, hoặc khi repo không có e2e. Là phần "làm thế nào" của [[verify-runtime-not-just-static-green]].
---

# bun-local-stack — Bật đúng phần cần, đẩy một việc đi hết đường

## Vì sao có skill này

Với stack có hàng đợi và worker, unit test xanh không chứng minh job chạy được: lỗi nằm ở
chỗ nối giữa các tiến trình (payload queue, worker chọn sai loại job, kết quả ghi sai cột,
UI không đọc lại). Repo không có e2e thì bấm thử bằng tay là phép thử duy nhất cho UI.

## Bước 1 — Tìm cách chạy từ chính repo

Đọc, không đoán:

- thư mục scripts (`dev*`, `start*`, `local-*`, `smoke-*`) và script ở gốc repo;
- scripts của `package.json` gốc và từng app (`dev`, `start`, các lệnh one-shot);
- file compose và file cấu hình hạ tầng local;
- mục "Running locally" trong README, cùng thư mục runbook/operations trong docs.

Ghi thành một bảng để dùng lại trong lượt này:

| Tiến trình | Lệnh | Cổng | Cần gì trước |
|------------|------|------|--------------|

README của một app mô tả framework khác với dependency thật trong `package.json` thì README
đó đã cũ: tin manifest, và báo user tên file tài liệu cũ.

Với workspace Bun: `bun run --filter=<tên-package> dev` chạy riêng một app; script one-shot
thường tự nạp env bằng `bun --env-file <file>`, đọc script để biết nó lấy env ở đâu.

## Bước 2 — Chọn lát cắt nhỏ nhất

| Thay đổi ở | Cần bật |
|------------|---------|
| UI | DB + API + web |
| API | DB + API |
| Worker / job | DB + queue + **một** worker đúng loại job + thứ kích job |
| Thiết bị, emulator, simulator, TV | các thứ trên + automation server, thiết bị đã boot, app test đã build và cài, proxy/CA nếu luồng có bắt traffic |

Bật cả stack khi chỉ cần hai tiến trình là phí thời gian và làm log khó đọc.

## Bước 3 — Tiền đề

- File env: tạo từ file mẫu của repo. Không commit, không in giá trị secret ra output.
- DB: migrate, seed nếu luồng cần dữ liệu. **Hỏi trước** khi chạy lệnh reset/drop.
- Cổng đang bị chiếm: tìm tiến trình giữ cổng, đừng đổi cổng trong code.
- Tiến trình chạy lâu: chạy nền, đọc log tới khi có dòng báo sẵn sàng rồi mới thử.
- Nền tảng cần máy riêng (macOS cho iOS, Windows cho một số emulator TV): không có thì ghi
  vào mục chưa verify, không giả lập bằng nền tảng khác.

## Bước 4 — Chạy đường thật

**UI**
1. Mở đúng trang, đi đúng luồng đã đổi, từ đường vào mà người dùng thật dùng.
2. Mở console và tab network: lỗi đỏ hoặc request 4xx/5xx ngoài dự kiến là fail.
3. Xem đủ trạng thái: đang tải, có dữ liệu, rỗng, lỗi.
4. Luồng có nhiều đường vào (từ danh sách, từ thông báo, từ link trực tiếp) thì thử từng đường.

**API / stream**
1. Gọi endpoint bằng token thật, đọc log của chính request đó.
2. SSE: `curl -N <url>` giữ kết nối, kích sự kiện ở cửa khác, xác nhận dòng `data:` về tới;
   ngắt `curl` rồi xem server có gỡ listener không.

**Job / worker**
1. Kích **một** job bằng UI hoặc lệnh one-shot của repo. Ghi lại id của nó.
2. Theo id đó qua từng chặng: bản ghi trong queue → log worker nhận job → dòng kết quả trong
   DB → artifact (file, video, log) → UI/API hiển thị lại.
3. Chặng nào đứt thì đó là chỗ lỗi; đọc log của chặng ngay trước nó.
4. Sửa hành vi có sẵn thì chạy cùng job trước và sau thay đổi để so.

**Thiết bị**
1. Chạy smoke script của repo trước, để tách lỗi môi trường khỏi lỗi code.
2. Smoke qua rồi mới chạy luồng đầy đủ. Mỗi nền tảng là một lần chạy riêng.

## Bước 5 — Dọn

Tắt tiến trình nền và container mình đã bật. Dữ liệu thử để lại trong DB local thì nêu
trong báo cáo nếu nó ảnh hưởng lần chạy sau.

## Bước 6 — Báo cáo

- Đã bật gì, bằng lệnh nào, mở URL nào.
- Đã quan sát gì ở từng chặng (trích log hoặc giá trị thật, không viết "chạy ổn").
- Cái gì **chưa** chạy được và vì sao: cần thiết bị thật, cần máy khác, cần credential, cần build.

## Không làm

- Trỏ local vào DB, queue hay bucket của staging/production để "thử cho nhanh".
- Sửa code hay cấu hình chỉ để chạy được ở máy mình rồi commit kèm.
- Suy kết quả của nền tảng này từ nền tảng khác.
