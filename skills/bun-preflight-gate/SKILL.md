---
name: bun-preflight-gate
description: Chạy ĐÚNG cổng kiểm của một repo Bun/TypeScript trước khi commit/push — tìm lệnh thật từ CLAUDE.md/README/package scripts/hook/CI thay vì đoán `npm test` hay `tsc`, chạy đủ theo thứ tự, tách lỗi mình gây ra khỏi lỗi có sẵn trên nhánh gốc, và báo từng lệnh kèm exit code. Dùng trước commit/push/mở MR, khi user nói "chạy pre-flight", "check trước khi push", "gate", "lint + test", hoặc khi một skill khác bảo chạy `npm test`/`tsc`/`flutter analyze` mà repo không dùng công cụ đó. Đi TRƯỚC [[bow-commit]], bổ trợ [[verify-runtime-not-just-static-green]].
---

# bun-preflight-gate — Chạy cổng kiểm của repo, không phải cổng mình quen tay

## Vì sao có skill này

Mỗi repo có package manager, type checker và test runner riêng. Lệnh quen tay của stack
khác hoặc không tồn tại, hoặc chạy được nhưng bỏ sót một nửa repo. Nhiều repo còn không có
job lint/test trên CI, khi đó cổng local là cổng duy nhất trước khi code vào nhánh chung.

## Bước 1 — Tìm lệnh thật

Đọc theo thứ tự này, nguồn đứng trước thắng nguồn đứng sau:

1. Mục "Pre-flight" / "Before push" / "Commands" trong `CLAUDE.md`, `CONTRIBUTING.md`, `README`.
2. Scripts của `package.json` gốc (`typecheck`, `lint`, `check`, `format`, `build`, `test`) và
   của từng package đã đổi.
3. Cấu hình hook (lefthook, husky): hook chạy gì thì đó là mức tối thiểu.
4. Cấu hình CI: job nào **thật sự tồn tại**, không phải job mà README nói là có.

Suy ra từ repo, không từ thói quen:

| Cần biết | Lấy ở đâu |
|----------|-----------|
| Package manager | lockfile (`bun.lock`) và trường `packageManager` |
| Type checker | script `typecheck`/`check`; có repo dùng `tsgo` thay `tsc` |
| Test runner | script `test` của **từng** package; một package có thể chạy Vitest trong khi phần còn lại chạy `bun test` |
| Phạm vi | `bunfig.toml`, `tsconfig` references, include/exclude của runner |

## Riêng với Bun

- Cài dependency khoá lockfile: `bun install --frozen-lockfile`.
- `bun run --filter='<mẫu>' <script>` chạy script của từng workspace. `bun test` ở gốc thì
  dùng runner của Bun và nghe theo mục `[test]` trong `bunfig.toml`: mục đó có thể loại hẳn
  một package (thường là web chạy Vitest). `bun test` xanh vì thế chưa nói gì về package bị
  loại; script `test` ở gốc mới là lệnh gọi đủ từng runner.
- `bunx <công-cụ>` chạy được công cụ không nằm trong dependency (tải tạm). Dùng nó khi tài
  liệu yêu cầu một công cụ mà manifest không có, và ghi rõ điều đó trong báo cáo.
- `tsgo --build` và `tsgo --noEmit -p .` phủ khác nhau. Lệnh gốc không gồm project của
  package mình đổi thì chạy thêm lệnh typecheck của package đó.
- `biome check .` chỉ kiểm; `biome check --write` và `biome format --write` là sửa file.

## Bước 2 — Xử lý chỗ các nguồn mâu thuẫn

- **Tài liệu yêu cầu công cụ không được cài** (không có trong manifest hay lockfile): vẫn thử
  chạy theo tài liệu bằng `bunx`, rồi báo "công cụ X không phải dependency, đã chạy
  on-demand". Không âm thầm bỏ bước đó.
- **Tài liệu nói CI chặn mà CI không có job tương ứng:** coi cổng local là duy nhất và báo user.
- **Lệnh gốc không phủ hết các package:** chạy thêm lệnh của package đã đổi.

## Bước 3 — Hook đã được cài chưa

```bash
git config core.hooksPath
ls .git/hooks | grep -v '\.sample$'
```

Cả hai đều trống thì hook pre-commit và commit-msg **không chạy** trên máy này. Khi đó tự
chạy tay phần hook lẽ ra làm: lint file đã stage, typecheck, và kiểm message bằng công cụ
commit-lint của repo trước khi commit. Báo user là hook chưa cài; không tự cài nếu chưa được yêu cầu.

## Bước 4 — Chạy, rẻ trước đắt sau

1. Cài dependency ở chế độ khoá lockfile.
2. Typecheck.
3. Lint.
4. Kiểm format (chế độ check, không ghi).
5. Build.
6. Test.

Trong monorepo: typecheck và lint chạy toàn repo; test tối thiểu là package đã đổi cộng các
package phụ thuộc nó; đổi package dùng chung thì chạy tất cả. Lệnh dài thì chạy foreground
với timeout đủ, đọc hết output, lấy exit code.

Không chạy auto-fix/format ghi đè trên cả repo để "cho qua". Nếu fix tự động đụng quá nửa
một file thì hỏi trước.

## Bước 5 — Lỗi có sẵn và lỗi của mình

Cổng đỏ chưa chắc do mình. Phân loại trước khi sửa:

```bash
git diff --name-only <nhánh-gốc>...HEAD      # file mình đã đụng
```

- Lỗi nằm trong file mình đụng → sửa hết, không ngoại lệ.
- Lỗi nằm ở file không đụng → kiểm nó có sẵn trên nhánh gốc (chạy cùng lệnh ở nhánh gốc
  bằng worktree, hoặc lọc output theo danh sách file). Có sẵn thì **không sửa lẫn** vào thay
  đổi này (mỗi commit một việc); ghi lại số lượng để báo.
- Test đỏ ở vùng mình không đụng → chạy lại riêng test đó một lần để loại flaky, rồi báo.

Không viết "gate pass" khi còn đỏ. Viết đúng trạng thái: "0 lỗi mới; N lỗi có sẵn ở file
không đụng, liệt kê bên dưới".

## Bước 6 — Báo cáo

Một bảng, mỗi lệnh một dòng:

| Lệnh | Exit | Ghi chú |
|------|------|---------|
| `<lệnh typecheck>` | 0 | |
| `<lệnh lint>` | 1 | 3 lỗi có sẵn ở file không đụng |
| `<lệnh test>` | 0 | 412 pass |

Bước nào không chạy được thì ghi lý do ở cột ghi chú, không bỏ dòng.

## Không làm

- `--no-verify`, bỏ qua hook, sửa cấu hình cổng để cho qua.
- Thêm comment tắt lint, `skip`/xoá test đang đỏ.
- Báo kết quả từ lần chạy trước khi sửa code. Sửa xong phải chạy lại.
