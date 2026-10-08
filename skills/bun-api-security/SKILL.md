---
name: bun-api-security
description: Soát bảo mật theo TỪNG BỀ MẶT của một backend Bun + Hono trước khi commit — JWT và cookie phiên, CORS và header, rate limit, phân quyền theo chủ sở hữu, SQL thô, URL do người dùng nhập (SSRF), upload và presign object storage, PII trong log, luồng xoá dữ liệu / consent / lưu giữ, secret và biến môi trường. Dùng khi sửa auth, middleware, cấu hình CORS, endpoint nhận file hoặc URL, logger, job dọn dữ liệu, file env; hoặc khi user nói "check bảo mật", "có lộ gì không", "an toàn chưa". Checklist cho một endpoint nằm ở [[bun-api-endpoint]]; threat model rộng ở [[security-and-hardening]].
---

# bun-api-security — Soát theo bề mặt, sửa ở chỗ repo đã có

## Bước 1 — Tìm chốt chặn repo đã có

Trước khi thêm gì, tìm và đọc: middleware auth, error handler chung, cấu hình CORS và secure
headers ở file entry, middleware rate limit, module đọc env, helper ẩn danh hoá log, helper
audit, module object storage. Sửa hoặc mở rộng tại đó. Viết một bản song song là tạo ra
đường thứ hai không ai soát.

## Bước 2 — Xác định thay đổi chạm bề mặt nào

Chỉ soát những dòng có liên quan trong bảng dưới, nhưng soát đủ từng ý của dòng đó.

| Bề mặt | Phải đúng |
|--------|-----------|
| **JWT** | Verify chữ ký bằng hàm verify của thư viện, cố định thuật toán; kiểm hạn; kiểm `iss` / `aud` nếu repo đặt. Không quyết định quyền từ token chỉ mới decode. Khoá lấy từ module env. Token không vào log, không vào URL sống lâu. |
| **Cookie phiên** | `HttpOnly`; `Secure` ở môi trường thật; `SameSite` tường minh; `Path` và hạn rõ ràng. `SameSite=None` hoặc client khác site thì mutation cần chống CSRF. Đăng xuất phải xoá cookie với đúng các thuộc tính đã đặt. |
| **CORS** | Origin lấy từ danh sách cho phép trong cấu hình. Không `*` và không phản chiếu `Origin` khi bật credentials. Thêm origin mới là thay đổi cần người duyệt. |
| **Header bảo mật** | Middleware secure headers đang bật thì giữ; nới một header phải có lý do ghi trong code. |
| **Rate limit** | Đăng nhập, đặt lại mật khẩu, endpoint đắt (xuất file, presign, kích job) đều có giới hạn. Khoá theo user khi đã đăng nhập. Khoá theo IP thì đọc IP đúng cách sau proxy, theo cấu hình hạ tầng của repo. |
| **Phân quyền** | Mỗi route có kiểm role; mỗi truy vấn theo id có lọc chủ sở hữu / tenant. Endpoint stream cũng phải xác thực. Route public cố ý thì đánh dấu như các route public sẵn có. |
| **SQL** | Giá trị luôn đi qua tham số. Tên cột, chiều sắp xếp, tên bảng đi qua `switch` hoặc allow-list trên giá trị đã validate. Hàm chạy SQL dạng chuỗi (`unsafe`) chỉ nhận SQL tĩnh của repo. |
| **URL do người dùng nhập** | Server sắp `fetch` một URL lấy từ request hoặc từ DB do người dùng ghi: chỉ `http` / `https`; chặn địa chỉ nội bộ, loopback, link-local và metadata của cloud sau khi phân giải DNS; giới hạn redirect, thời gian, kích thước phản hồi. |
| **Upload / presign** | Key do server sinh, có tiền tố của thực thể đã kiểm quyền; không presign theo key client gửi. Hạn ngắn. Giới hạn content-type và kích thước. Presign đọc chỉ cấp sau khi kiểm quyền đọc. Bucket không mở công khai. |
| **Log** | Logger của request, không `console`. Không email, tên, số điện thoại, IP, token, nội dung body. Lỗi DB có thể kèm giá trị: ẩn danh hoá trước khi log. Audit log theo đúng mức repo đã quy định. |
| **Dữ liệu cá nhân** | Xoá theo yêu cầu phải đi hết: mọi bảng tham chiếu, file trên object storage, bản sao trong audit. Consent được kiểm ở server. Job lưu giữ / dọn dữ liệu có test và chạy được lại. |
| **Secret và env** | Chỉ đọc env qua module env của repo. Không secret trong diff, log, hay lỗi trả về client. Biến được build vào bundle của web (tiền tố công khai của bundler) là công khai: không đặt secret vào đó. |
| **Dependency** | Thêm package thì đối chiếu danh sách cho phép / cấm của repo trước. Chạy lệnh audit của package manager nếu phiên bản đang dùng có. |

## Bước 3 — Test là một phần của thay đổi

Đường nào trong bảng trên bị đổi thì có test đi cùng commit:

- auth: thiếu token, token hết hạn, token sai chữ ký, sai role, không phải chủ sở hữu;
- validate: mỗi trường biên, và input cố ý độc hại cho đúng bề mặt đó (URL nội bộ, key có `../`);
- mutation: tác dụng phụ đúng, audit có ghi;
- luồng xoá / consent / lưu giữ: dữ liệu thật sự biến mất ở mọi nơi đã liệt kê.

## Bước 4 — Verify runtime

Gọi server đang chạy với user thật, sai quyền, và input độc hại đã test. Với CORS và cookie,
kiểm bằng trình duyệt từ đúng origin của web, vì `curl` không áp các luật đó. Bề mặt nào
chưa chạy thật thì ghi "chưa verify" ([[verify-runtime-not-just-static-green]]).

## Chặn cứng — dừng và báo, không commit

- Secret, token hoặc credential trong diff.
- Endpoint mới không có kiểm auth / authz.
- Đường auth, mutation, phân quyền hoặc xoá dữ liệu chưa có test.
- PII trong log.
- Nới CORS, tắt header bảo mật, hoặc gỡ rate limit mà chưa có người duyệt.

## Báo cáo

- Bề mặt đã soát, từng bề mặt một dòng: đạt / đã sửa gì / chưa verify vì sao.
- Chỗ repo đang lệch với bảng trên nhưng nằm ngoài thay đổi này: liệt kê để user quyết, không sửa lẫn.
