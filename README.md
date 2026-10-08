# bow-skill-bun-api

Bộ skill cho backend **TypeScript chạy trên Bun với Hono**, dùng với [bow-gid](https://github.com/Bow-T/bow-gid).
Theo hướng **mỗi công nghệ một repo**: một dự án tick bộ này CÙNG với các bộ khác của mình (Supabase,
Stripe, bộ dùng chung "Quy trình làm việc"…) ở ô **Công nghệ**.

Bộ này không chứa kiến thức của sản phẩm nào. Mỗi skill bắt đầu bằng việc tìm quy ước từ chính repo đang
làm (route anh em, runner migration, script gate), rồi mới áp checklist.

## Cấu trúc

```
bow-skill-bun-api/
├── bow-skill.json        # manifest: id "bun-api" + quy tắc tự nhận diện repo
└── skills/
    ├── bun-preflight-gate/SKILL.md
    ├── bun-api-endpoint/SKILL.md
    ├── bun-api-security/SKILL.md
    ├── bun-db-migration/SKILL.md
    ├── bun-queue-worker/SKILL.md
    ├── bun-local-stack/SKILL.md
    └── ts-union-variant/SKILL.md
```

| Skill | Dùng khi |
| --- | --- |
| `bun-preflight-gate` | trước commit / push / mở MR: tìm và chạy đúng typecheck, lint, format, build, test của repo; tách lỗi có sẵn khỏi lỗi mới |
| `bun-api-endpoint` | thêm hoặc sửa một route Hono: auth, validate, DB, lỗi, audit, log, và 6 ca test tối thiểu |
| `bun-api-security` | sửa auth, CORS, cookie, endpoint nhận file hoặc URL, logger, luồng xoá dữ liệu, file env: soát theo từng bề mặt |
| `bun-db-migration` | thêm hoặc sửa migration (Drizzle + SQL viết tay): tìm cơ chế nạp, đăng ký đủ chỗ, idempotent, apply hai lần |
| `bun-queue-worker` | thêm hoặc sửa job qua hàng đợi và worker: payload tương thích ngược, handler chạy lại được, hạn hiển thị, job độc, tắt êm |
| `bun-local-stack` | chạy thử trên stack nhiều tiến trình (DB + API + web + worker + thiết bị): bật lát cắt nhỏ nhất, đẩy một job đi hết đường |
| `ts-union-variant` | thêm / bỏ / đổi tên một giá trị của union hay enum dùng chung giữa API, web, worker, app native |

Các skill trỏ sang skill của bộ khác bằng `[[tên]]` (`impact-sweep`, `verify-runtime-not-just-static-green`,
`test-driven-development`, `security-and-hardening` ở bộ `process`; `bow-commit` ở bộ `commit`). Dự án không
tick các bộ đó thì các skill ở đây vẫn dùng được, chỉ thiếu phần trỏ sang.

## Tự nhận diện

bow-gid tự tick bộ này cho dự án chưa khai công nghệ khi repo khớp một trong các quy tắc ở `targets.detect`:

| Quy tắc | Nghĩa |
| --- | --- |
| `package.json` chứa `"hono"` | có package dùng Hono (tìm cả trong thư mục con của monorepo) |
| có `bunfig.toml` | repo chạy trên Bun |

Chỉ soi file git đã track. Repo Bun không có HTTP API (CLI, thư viện) cũng khớp quy tắc thứ hai; khi đó bỏ
tick ở ô Công nghệ.

## Đăng ký vào bow-gid

Thêm vào `stacks` của `~/.bow-gid/registry.json` (bow chỉ tải repo có trong danh sách này, đúng tag đã ghim):

```json
{ "id": "bun-api", "label": "Bun API (Hono)", "repo": "github.com/Bow-T/bow-skill-bun-api", "ref": "v1.0.0" }
```

Thử local trước khi đẩy GitHub: `"repo": "file:///đường/dẫn/tới/bow-skill-bun-api"`.

## Phát hành phiên bản mới

Sửa qua PR → tăng `version` trong `bow-skill.json` → gắn tag (`git tag v1.0.1 && git push --tags`) → cập
nhật `ref` trong registry. bow tải theo tag và cache theo `<id>@<ref>`, nên sửa nội dung mà giữ nguyên tag sẽ
KHÔNG được tải lại.

Giữ mô tả (`description`) của mỗi skill dưới 1024 ký tự: dài hơn thì Claude Code bỏ mô tả, chỉ còn tên.
