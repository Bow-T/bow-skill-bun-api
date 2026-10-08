---
name: bun-db-migration
description: Thêm hoặc sửa migration DB (Drizzle + SQL viết tay) sao cho nó THẬT SỰ chạy ở mọi môi trường — tìm ra cơ chế nạp migration của repo (journal của ORM, quét thư mục, hay danh sách đăng ký tay trong runner), đánh số đúng, viết idempotent khi runner chạy lại file, đồng bộ schema ORM với SQL, rồi apply trên DB thật và chạy lại lần hai. Dùng khi thêm/sửa cột, bảng, index, constraint, trigger, enum DB, queue; khi user nói "thêm migration", "đổi schema", "add column"; hoặc khi một migration "đã merge mà DB không đổi". Bổ trợ [[impact-sweep]] (mọi reader của cột) và [[verify-runtime-not-just-static-green]].
---

# bun-db-migration — Migration phải được nạp, không chỉ được viết

## Vì sao có skill này

File SQL đúng cú pháp mà runner không nạp thì không bao giờ chạy. Typecheck, lint và test
có mock DB đều xanh; lỗi chỉ lộ ra trên môi trường thật, sau khi đã merge. Mỗi repo nạp
migration một kiểu, nhiều repo trộn hai kiểu, và tài liệu về nó thường đã cũ. Vì vậy đừng
đoán: đọc cơ chế từ chính code.

## Bước 0 — Xác nhận trước khi viết

Migration là thay đổi khó đảo. Trước khi tạo file, cho user xem: bảng/cột nào đổi, dữ liệu
hiện có bị ảnh hưởng thế nào, có khoá bảng lớn không, đường lùi là gì. Được đồng ý rồi mới viết.

## Bước 1 — Tìm cơ chế nạp (bắt buộc, làm bằng grep)

Lấy tên file migration **mới nhất** rồi grep nó khắp repo (bỏ `node_modules`, thư mục build):

```bash
ls <thư-mục-migrations> | sort | tail -3
grep -rn "<tên-file-mới-nhất-không-đuôi>" . --exclude-dir=node_modules --exclude-dir=dist
```

Mỗi hit ngoài chính file đó là một nơi migration mới cũng phải xuất hiện. Kết quả rơi vào
một trong ba kiểu:

| Kiểu | Dấu hiệu | Việc phải làm |
|------|----------|---------------|
| **Journal do tool sinh** | hit trong `meta/_journal.json` hoặc snapshot của ORM | Sinh migration bằng lệnh generate của repo. Không sửa tay journal. |
| **Runner quét thư mục** | không hit nào; runner có `readdir`/glob | Chỉ cần đặt đúng tên và số thứ tự. |
| **Đăng ký tay** | hit trong file runner (hằng đường dẫn, `readFileSync`, danh sách) | Lặp lại **mọi** hit của file gần nhất cho file mới. Thường là hai chỗ: khai báo đường dẫn và khối apply. |

Kiểm thêm ba điểm:

- **Repo trộn kiểu.** Một phần migration đi qua journal, phần viết tay đi qua runner. Xem
  migration gần nhất *cùng loại với cái sắp viết* đi đường nào, đừng lấy file số 0001 làm mẫu.
- **Local và deploy có dùng cùng một runner không.** Có repo chạy CLI của ORM ở local nhưng
  deploy chạy một script riêng. Runner của deploy mới là chuẩn để verify.
- **Tài liệu migration có khớp code không.** README cũ thì ghi lại để báo user, không làm theo nó.

## Riêng với Drizzle

- `drizzle-kit generate` sinh file SQL **và** ghi `meta/_journal.json` cùng snapshot. Hàm
  `migrate()` của `drizzle-orm` chỉ chạy file có tên trong journal: file SQL viết tay đặt
  cạnh đó mà không có trong journal sẽ bị bỏ qua, trừ khi script runner của repo tự đọc và
  chạy nó.
- CLI `drizzle-kit migrate` và hàm `migrate()` có thể ghi lịch sử vào **hai bảng khác nhau**
  (bảng mặc định của migrator so với bảng khai ở `migrations` trong `drizzle.config`). Một
  migration "đã chạy" ở local bằng CLI chưa chắc đã được runner của deploy coi là đã chạy.
- Schema khai `casing: 'snake_case'` thì field TypeScript là camelCase còn cột SQL là
  snake_case. Đối chiếu tên cột trong SQL với kết quả ánh xạ, không với tên field.
- Thứ cần có trước schema (extension, queue) nằm ở migration số nhỏ hơn. Giữ thứ tự đó.

## Bước 2 — Số thứ tự

- Số kế tiếp = số lớn nhất + 1, tính **sau khi fetch nhánh tích hợp**. Nhánh song song có
  thể đã lấy số đó; trùng số là lỗi thứ tự apply.
- Không đổi số, không đổi tên file đã có.
- Migration đã chạy ở môi trường dùng chung thì không sửa nội dung. Viết migration mới để chỉnh.

## Bước 3 — Viết SQL

- **Idempotent khi runner chạy lại file mỗi lần deploy** (đọc runner để biết): `IF NOT EXISTS`,
  `CREATE OR REPLACE`, `DROP ... IF EXISTS`, hoặc khối `DO $$ ... $$` có điều kiện. Ghi một dòng
  comment đầu file nêu vì sao nó an toàn khi chạy lại, theo mẫu các file gần nhất.
- **Cột `NOT NULL` trên bảng đã có dữ liệu:** thêm kèm `DEFAULT`, hoặc thêm nullable → backfill → siết lại.
- **Constraint/unique mới:** kiểm dữ liệu hiện có trước; có bản ghi vi phạm thì migration phải xử lý chúng.
- **Index trên bảng lớn:** `CONCURRENTLY`, và nhớ nó không chạy được trong transaction.
- **Đổi tên hoặc đổi nghĩa cột khi code cũ còn chạy lúc deploy:** tách ba bước mở rộng →
  chuyển dữ liệu → thu hẹp, mỗi bước một lần release.
- **`CREATE OR REPLACE` cả thân function/trigger:** lấy thân mới nhất trong migrations làm
  gốc, không lấy bản mình nhớ; bản cũ sẽ xoá mất thay đổi của người khác.

## Bước 4 — Đồng bộ những thứ đi cùng schema

- File schema của ORM khớp SQL: tên cột và casing, nullability, default, kiểu.
- Type dùng chung, schema validate (Zod…), serializer.
- Seed, reset script, fixture và mock dữ liệu cho test.
- Mọi nơi **đọc** cột/bảng đó, không chỉ nơi ghi → [[impact-sweep]]. Bảng có soft delete thì
  query mới phải lọc cột xoá mềm.

## Bước 5 — Verify trên DB thật

1. Apply bằng runner của deploy trên DB local. Đọc log: tên file mới phải xuất hiện.
2. Chạy runner **lần thứ hai**. Phải qua, không lỗi "already exists".
3. Thử trên cả DB trống (dựng từ đầu) và DB đã có dữ liệu.
4. Query trực tiếp: cột, default, constraint, index có đúng như định viết.
5. Chạy test của các package đụng tới bảng đó.

Không có DB để chạy thì nói thẳng: "SQL đã viết và đã đăng ký, CHƯA apply trên DB thật vì <lý do>".

## Báo cáo

- File migration và số thứ tự.
- Đã đăng ký ở những đâu (liệt kê từng chỗ, kèm kiểu nạp đã xác định ở Bước 1).
- Đã apply mấy lần, trên DB nào, kết quả.
- Đường lùi nếu cần gỡ.

## Thấy mình nghĩ thế này thì dừng

| Ý nghĩ | Thực tế |
|--------|---------|
| "File nằm đúng thư mục là chạy" | Chỉ đúng với runner quét thư mục. Kiểm Bước 1. |
| "Test xanh rồi" | Test mock DB không chạy SQL của bạn. |
| "Sửa luôn file migration cũ cho gọn" | Môi trường đã chạy file đó sẽ không chạy lại, hoặc chạy lại và hỏng. |
| "Chạy một lần được là xong" | Runner chạy lại file thì lần hai mới là phép thử. |
