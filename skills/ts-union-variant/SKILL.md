---
name: ts-union-variant
description: Thêm (hoặc bỏ, đổi tên) MỘT giá trị của union/enum TypeScript dùng chung giữa API, web, worker và app native — platform, action, status, role, loại job — sao cho không sót chỗ nào: chỗ compiler tự báo, chỗ compiler KHÔNG báo (validator, danh sách tuỳ chọn UI, CHECK của DB, code native chép tay), bảng năng lực theo từng giá trị, và dữ liệu đã lưu theo luật cũ. Dùng khi user nói "thêm platform", "thêm action", "thêm status", "thêm loại mới", hoặc khi sửa một `type X = 'a' | 'b'` / `z.enum([...])` có nhiều nơi dùng. Là công thức riêng cho TypeScript của [[impact-sweep]]; không thay skill đó.
---

# ts-union-variant — Thêm một giá trị, sửa đủ mọi nơi liệt kê các giá trị còn lại

## Vì sao có skill này

Grep tên giá trị mới không ra gì, vì chưa nơi nào chứa nó. Những nơi cần sửa là nơi đang
liệt kê các giá trị **cũ**. TypeScript chỉ chỉ ra một phần trong số đó.

## Bước 1 — Tìm nguồn gốc của union

Tìm nơi khai báo, thường ở package dùng chung. Một union hay có ba hình dạng song song, và
cả ba phải có giá trị mới:

- type: `type Platform = 'a' | 'b'`;
- mảng hằng: `const PLATFORMS = ['a', 'b'] as const`;
- validator: `z.enum([...])` hoặc một nhánh của `z.discriminatedUnion(...)`.

Type suy từ mảng hay từ schema (`z.infer`) thì chỉ cần sửa một nơi. Ba nơi viết tay riêng
thì sửa cả ba, và đề xuất gộp về một nguồn như một việc riêng.

## Bước 2 — Lấy một giá trị anh em làm mẫu rồi grep nó

Chọn giá trị đã chạy đầy đủ và giống giá trị mới nhất. Grep chuỗi của nó trên **toàn repo**,
không riêng thư mục TypeScript:

```bash
grep -rn "'<giá-trị-anh-em>'\|\"<giá-trị-anh-em>\"" . --exclude-dir=node_modules --exclude-dir=dist
```

Mỗi hit là một dòng trong checklist. Trình danh sách này cho user trước khi sửa.

## Bước 3 — Phân loại từng chỗ

**Compiler sẽ báo** (sửa theo lỗi typecheck):

- `Record<Union, …>` và bảng có `satisfies Record<Union, …>`;
- `switch` có nhánh `default` gán `const _exhaustive: never = x`;
- object literal bắt buộc đủ key theo union.

**Compiler KHÔNG báo** (phải tự tìm):

- `switch` có `default` thường, chuỗi `if (x === 'a' || x === 'b')`, toán tử ba ngôi;
- `['a', 'b'].includes(x)` và các allow-list khác;
- `Partial<Record<Union, …>>` và map có fallback;
- validator khai riêng với type;
- danh sách tuỳ chọn, nhãn, icon, màu ở UI; bộ lọc ở trang danh sách;
- ràng buộc `CHECK`, kiểu enum, `IN (...)` trong SQL và trong function/trigger của DB;
- tên queue, khoá định tuyến job, chỗ worker chọn engine theo giá trị;
- app native hoặc script ngoài TypeScript chép lại danh sách bằng tay;
- seed, fixture, bảng `it.each`, tài liệu liệt kê giá trị.

## Bước 4 — Bảng năng lực theo từng giá trị

Repo thường có bảng "giá trị nào hỗ trợ cái gì" (action nào chạy được trên platform nào,
role nào được làm gì). Với giá trị mới:

- điền **từng** bảng một cách có chủ đích; chép nguyên dòng của giá trị anh em là đoán;
- chỗ chưa chắc thì hỏi, và để giá trị mới ở phía chặt (không hỗ trợ) cho tới khi có câu trả lời;
- bảng ở package dùng chung thường được cả API (validate), worker (thực thi) và UI (ẩn/hiện)
  đọc. Kiểm cả ba nơi dùng cùng một bảng, không có bản chép riêng.

## Bước 5 — Dữ liệu đã lưu

Bản ghi lưu trước thay đổi đã được validate theo luật cũ.

- Thêm giá trị: bản ghi cũ vẫn hợp lệ, nhưng mọi chỗ đọc ra rồi `switch` phải chịu được cả giá trị mới.
- Bỏ hoặc đổi tên giá trị: cần migration dữ liệu, và chịu được cả hai tên trong lúc deploy.
- Đổi năng lực của một giá trị: validate lại dữ liệu đã lưu ở mọi đường đọc lại nó. Một lần
  update chỉ đổi bản ghi cha vẫn phải kiểm lại các bản ghi con đã lưu theo cha cũ.
- Validate hai tầng (hình dạng trước, đối chiếu với giá trị của bản ghi cha sau) thì giá trị
  mới phải đi qua **cả hai** tầng, ở create, ở update, và ở update chỉ đổi cha.

## Bước 6 — Để lần sau compiler tự báo

Gặp chỗ compiler không báo mà chuyển được thì chuyển, theo đúng kiểu repo đang dùng:

- `switch` → thêm nhánh `default` với `const _exhaustive: never = x` (hoặc helper `assertNever`
  nếu repo đã có; đừng thêm helper mới khi repo đang dùng kiểu inline);
- map → `satisfies Record<Union, …>`;
- test → một test lặp qua mảng hằng của union, assert mỗi giá trị đều được xử lý.

Việc chuyển này là refactor: tách commit riêng, không trộn vào commit thêm giá trị.

## Bước 7 — Verify

1. Typecheck **mọi** project. Lệnh gốc có thể không phủ hết package; chạy thêm lệnh của package còn thiếu.
2. Test của package dùng chung và của từng nơi dùng.
3. Runtime: tạo một bản ghi mang giá trị mới qua API thật → worker xử lý → UI hiển thị đúng
   nhãn. Tầng nào chưa chạy được thì ghi "chưa verify" kèm lý do
   ([[verify-runtime-not-just-static-green]]).

## Báo cáo

- Số chỗ đã sửa, chia hai nhóm: compiler báo / tự tìm.
- Từng bảng năng lực đã điền gì cho giá trị mới, chỗ nào đang để ở phía chặt và chờ ai trả lời.
- Dữ liệu đã lưu: có cần migration không, đã làm chưa.
- Tầng runtime nào đã chạy thật.
