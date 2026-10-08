---
name: bun-api-endpoint
description: Thêm hoặc sửa một HTTP endpoint Hono theo đúng khuôn của repo — chép cấu trúc từ route "anh em" gần nhất thay vì tự nghĩ, đủ 6 lớp (auth/authz, validate ở biên bằng schema dùng chung, truy cập DB theo idiom sẵn có, map lỗi về mã chuẩn, audit cho mutation, không lộ PII ra log) và kèm test theo harness test route có sẵn. Dùng khi tạo route/handler mới, đổi request/response shape, thêm query param/filter/sort, hoặc khi user nói "thêm API", "thêm endpoint", "sửa route". Không thay [[security-and-hardening]] (threat model rộng): skill này là checklist cho từng endpoint. Bổ trợ [[impact-sweep]] (mọi client gọi endpoint) và [[test-driven-development]].
---

# bun-api-endpoint — Một endpoint, sáu lớp, theo khuôn có sẵn

## Bước 1 — Tìm route anh em và đọc trọn

Chọn route gần nhất: cùng resource, hoặc cùng loại thao tác (list / detail / create / update /
delete). Đọc hết file route **và file test của nó**. Ghi ra trước khi viết:

- thứ tự chuỗi middleware (auth → validate → handler);
- thư viện và helper validate, cách validate path param;
- cách lấy user hiện tại và role;
- idiom truy cập DB (query builder hay SQL thô có tham số), có transform tên cột không;
- lớp lỗi và error handler chung, shape JSON của lỗi;
- helper audit, logger theo request;
- comment đầu file (danh sách endpoint, mã ticket) nếu repo có thói quen đó.

Code mới phải đọc như code của file đó. Không mang khuôn của framework khác vào.

## Khuôn Hono thường gặp

```ts
const app = new Hono<AppEnv>();

app.get('/', requireAuth, zValidator('query', listQuerySchema), async (c) => {
  const query = c.req.valid('query');
  const user = c.get('user');
  // ...
  return c.json(rows);
});

export default app;
```

- Một file route là một `Hono` con, mount ở file entry. Thêm file route mới thì nhớ mount.
- Dữ liệu đã validate lấy bằng `c.req.valid(...)`; đừng đọc lại `c.req.json()` / `c.req.query()`.
- Biến theo request (user, logger, request id) nằm ở `c.get(...)`, kiểu khai trong generic `Env`.
- Lỗi: `HTTPException` hoặc lớp lỗi riêng của repo, map ở `app.onError`. Tên và import cụ thể
  lấy từ route anh em, không lấy từ đoạn mẫu này.

## Bước 2 — Sáu lớp phải có

| Lớp | Phải đúng |
|-----|-----------|
| **1. Auth / authz** | Mọi route mới có middleware auth. Route cố ý public thì đánh dấu và ghi lý do theo cách các route public sẵn có đang làm. Phân quyền gồm cả role **và** quyền sở hữu: lọc theo owner/tenant ngay trong query, không chỉ kiểm role (chống đọc bản ghi của người khác qua id). |
| **2. Validate ở biên** | Body, query, path param đều qua schema. Schema mà client cũng dùng thì đặt ở package dùng chung. Sort/order/filter dạng tên cột phải qua allow-list rồi mới ghép vào query. Phân trang có trần `limit`. |
| **3. DB** | Theo idiom của repo, luôn tham số hoá. Bảng có soft delete thì lọc cột xoá mềm. Kết quả một dòng có thể rỗng (`rows[0]` là `T \| undefined` khi bật `noUncheckedIndexedAccess`) → trả 404. Nhiều lệnh ghi liên quan → một transaction. Vi phạm unique/FK → map sang 409/400 bằng helper có sẵn. |
| **4. Lỗi** | Ném lớp lỗi chung, để error handler map sang mã và shape thống nhất. Không trả message thô của DB hay stack trace. |
| **5. Mutation** | Ghi audit bằng helper có sẵn (ai, thực thể nào, hành động, thay đổi). Đẩy job/event **sau khi** ghi DB thành công. Client có thể retry thì thao tác phải an toàn khi lặp. |
| **6. Log** | Dùng logger của request, không `console`. Không ghi email, tên, số điện thoại, IP, token. Lỗi DB có thể chứa giá trị người dùng: ẩn danh hoá trước khi log, bằng helper của repo nếu có. |

## Bước 3 — Hợp đồng với client

- Đổi shape request/response → sửa type và schema dùng chung, rồi mọi client gọi nó
  ([[impact-sweep]]). Thêm field tuỳ chọn thì an toàn; đổi tên, đổi kiểu, bỏ field là thay đổi contract.
- Endpoint stream (SSE/WebSocket): dùng cơ chế fan-out có sẵn của repo. Không mở một kết nối
  DB hay một listener cho mỗi client.
- Endpoint mà worker/service khác gọi: kiểm cách nó xác thực, đừng vô tình chặn nó khi thêm auth.

## Bước 4 — Test theo harness có sẵn

Chép phần setup từ test của route anh em: cách mock env, DB, logger; cách ký token; cách gọi app.

- Hono test được không cần server: `await app.request('/path', { method, headers, body })`
  rồi assert `res.status` và `await res.json()`. Gắn error handler của repo vào app test, nếu
  không lỗi sẽ ra 500 thay vì mã thật.
- `mock.module(...)` của `bun:test` phải đứng **trước** `await import('./route')`.
- Dùng helper mock **đầy đủ** mà repo đã có. Module mock thiếu export làm hỏng những file
  test khác import cùng module, nhất là khi runner không cô lập từng file.

Tối thiểu mỗi endpoint:

1. Không có token → 401.
2. Sai role hoặc không phải chủ sở hữu → 403 (hoặc 404 nếu repo che sự tồn tại).
3. Input sai, từng trường biên → 400.
4. Không tồn tại → 404.
5. Đường chính → đúng status, đúng shape, **và** đúng tác dụng phụ (query đã ghép, audit, job).
6. Xung đột (trùng unique, trạng thái không cho phép) → 409.

Quy ước của repo đòi integration test trên DB thật mà test route hiện có lại mock DB: làm
theo pattern của file anh em và báo user chỗ mâu thuẫn. Không tự đổi cách test của cả repo.

## Bước 5 — Verify runtime

Gọi endpoint trên server đang chạy bằng token của user thật: một lần đúng quyền, một lần
sai quyền, một lần input biên. Đọc log server của chính request đó. Chưa gọi được thì báo
"test route xanh, CHƯA gọi trên server thật" ([[verify-runtime-not-just-static-green]]).

## Chặn cứng — không commit khi còn

- Endpoint mới không có kiểm auth/authz.
- Mutation, đường auth hoặc đường phân quyền chưa có test.
- PII hoặc secret trong log.
- Ghép chuỗi lấy từ request vào SQL.
