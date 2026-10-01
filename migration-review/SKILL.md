---
name: migration-review
description: Use when creating or editing a database migration file to assess risk, ask clarifying questions, require confirmation for high-risk changes, and document the review.
---

# Migration Review

## Khi nào dùng

Skill này chạy TỰ ĐỘNG (qua hook) mỗi khi Claude tạo hoặc chỉnh sửa file `db/migrations/*.js`.
Cũng có thể gọi thủ công bằng `/migration-review`.

**HARD RULE:** Không được bỏ qua skill này khi làm việc với migration files.

---

## Bước 1 — Phân tích file migration

Đọc file migration vừa tạo/chỉnh sửa. Tự động xác định:

| Thuộc tính | Cách phát hiện |
|-----------|---------------|
| Có data migration? | File chứa INSERT, UPDATE, DELETE, hoặc `sequelize.query` với data-modifying SQL |
| Ảnh hưởng permissions? | JOIN sang bảng `user` + filter theo `role_id`, `is_active`, permission fields; hoặc bảng liên quan access control |
| Có rollback? | `down()` có nội dung thực sự hay chỉ `return Promise.resolve()` |
| Schema only? | Chỉ có `addColumn`, `removeColumn`, `createTable`, `dropTable`, `addIndex` |

---

## Bước 2 — Hỏi 5 câu (bỏ qua câu đã rõ từ code)

Hỏi **từng câu một**, đợi trả lời trước khi hỏi tiếp. Nếu đã biết câu trả lời từ phân tích code, bỏ qua câu đó.

**Câu 1 — Môi trường:** Migration này sẽ chạy trên môi trường nào?
- A. Chỉ dev/local
- B. Dev + staging
- C. Dev + staging + production

**Câu 2 — Loại migration:** Migration có INSERT/UPDATE/DELETE data không, hay chỉ thay đổi schema (ADD COLUMN, CREATE TABLE...)?
- A. Schema only (ADD COLUMN, CREATE TABLE, ADD INDEX...)
- B. Có data migration (INSERT/UPDATE/DELETE)
- C. Cả hai

*(Nếu câu 2 = A → bỏ qua câu 3 và 4, không có Bước 1.5)*

*(Nếu câu 2 = B hoặc C → chạy **Bước 1.5** trước khi hỏi câu 3)*

*(Câu 3 chỉ hỏi nếu Bước 1.5 không xác định được phạm vi rõ ràng — xem điều kiện fallback ở Bước 1.5)*

**Câu 3 — Phạm vi ảnh hưởng:** Ai/cái gì bị ảnh hưởng bởi data migration này?
- A. Toàn bộ user/data trong hệ thống
- B. Một số enterprise hoặc user cụ thể
- C. Bảng nhỏ, ít dữ liệu, không ảnh hưởng user trực tiếp

**Câu 4 — Permissions/Visibility:** Migration có ảnh hưởng đến quyền truy cập hoặc data visibility của user không?
- A. Có — user có thể thấy data mới hoặc mất quyền xem
- B. Không rõ
- C. Không

**Câu 5 — Rollback:** `down()` có rollback được không?
- A. Có — rollback đầy đủ
- B. Partial — rollback một phần (vd: xóa column nhưng không restore data)
- C. Không — `down()` là no-op

---

---

## Bước 1.5 — Phân tích bảng bị ảnh hưởng *(chỉ chạy khi Câu 2 = B hoặc C)*

### Cách phân tích

Đọc toàn bộ nội dung migration file, xác định:

| Thông tin | Cách phát hiện |
|-----------|---------------|
| Tên bảng | Tham số đầu tiên của `queryInterface.*()`, tên bảng trong raw SQL (`INTO`, `FROM`, `UPDATE`, `JOIN`) |
| Loại thao tác | `INSERT`, `UPDATE`, `DELETE` → data-modifying; `SELECT`, `JOIN` → READ |
| Điều kiện lọc | Mệnh đề `WHERE`, điều kiện `JOIN ON`, tham số `where:` trong Sequelize |
| Ước tính phạm vi | Không có WHERE → "toàn bộ"; có WHERE filter role/enterprise → "subset cụ thể"; JOIN với nhiều bảng → ghi rõ nguồn |

### Present kết quả

```
📊 Phân tích bảng bị ảnh hưởng (tự động):

| Bảng | Thao tác | Điều kiện | Ước tính phạm vi |
|------|----------|-----------|-----------------|
| ten_bang | INSERT | WHERE x = 1 | Subset — chỉ rows thỏa điều kiện |
| bang_nguon | READ (JOIN) | JOIN ON ... | Chỉ đọc, không thay đổi |
```

### 3 câu xác nhận phạm vi (hỏi từng câu, đợi trả lời)

**Câu xác nhận 1/3 — Phạm vi tổng:**
```
Migration này ảnh hưởng toàn bộ data hay chỉ subset theo điều kiện?
A. Toàn bộ — không có filter đặc biệt ngoài JOIN
B. Subset — chỉ rows thỏa điều kiện cụ thể (enterprise X, role Y...)
```

**Câu xác nhận 2/3 — Nguồn cập nhật:**
```
Data được INSERT/UPDATE dựa trên data hiện có (có lọc nguồn) hay không lọc?
A. Có lọc nguồn — chỉ lấy data thỏa WHERE/JOIN điều kiện
B. Không lọc — INSERT/UPDATE toàn bộ không phân biệt
```

**Câu xác nhận 3/3 — Insert rows mới:**
```
Migration này có tạo ra rows mới trong DB không?
A. Có — INSERT thêm rows mới
B. Không — chỉ UPDATE rows đã có
```

### Câu 3 fallback

Sau 3 câu xác nhận trên, chỉ hỏi Câu 3 gốc nếu:
1. **Phân tích không xác định được bảng đích:** SQL dùng tên bảng động (string concatenation, biến), hoặc gọi stored procedure.
2. **Developer chọn "B. Subset" ở câu xác nhận 1/3 nhưng điều kiện trong code không rõ enterprise/user cụ thể** — cần hỏi thêm để ghi vào checklist.

Nếu không rơi vào 2 trường hợp trên → bỏ qua Câu 3, tiếp tục Câu 4.

---

## Bước 2.5 — Điều chỉnh SQL migration theo câu trả lời

**Bước này bắt buộc chạy trước khi tạo hoặc chỉnh sửa file migration.**

Dùng câu trả lời từ Bước 1.5 để quyết định SQL cụ thể:

### Câu xác nhận 3/3 — Insert rows mới

| Trả lời | SQL phải dùng |
|---------|--------------|
| A — Có INSERT rows mới | `INSERT INTO ... SELECT ... WHERE NOT EXISTS (...)` |
| B — Chỉ UPDATE rows đã có | `UPDATE ... SET ... WHERE ...` (không dùng INSERT) |

### Câu xác nhận 1/3 — Phạm vi tổng

| Trả lời | SQL phải có |
|---------|------------|
| A — Toàn bộ | Không cần filter đặc biệt ngoài JOIN điều kiện cơ bản |
| B — Subset | Phải có WHERE / JOIN điều kiện cụ thể thu hẹp phạm vi |

### Câu xác nhận 2/3 — Nguồn cập nhật

| Trả lời | SQL phải có |
|---------|------------|
| A — Có lọc nguồn | JOIN với bảng nguồn + điều kiện WHERE/ON cụ thể |
| B — Không lọc | SELECT/UPDATE không qua bảng nguồn trung gian |

### Nếu câu trả lời mâu thuẫn với SQL đã có trong file

Nếu file migration đã tồn tại và SQL không khớp với câu trả lời, hỏi developer:
```
SQL hiện tại dùng [INSERT/UPDATE] nhưng bạn trả lời [câu trả lời].
Cập nhật SQL theo câu trả lời không? (Y/N)
```
Nếu Y → chỉnh sửa SQL. Nếu N → ghi chú vào checklist và tiếp tục.

---

## Bước 3 — Tính Risk Level

```
HIGH   = (câu 2 = B hoặc C) VÀ (câu 4 = A hoặc B) VÀ (câu 5 = B hoặc C)
MEDIUM = (câu 2 = B hoặc C) HOẶC (câu 5 = B hoặc C)
LOW    = câu 2 = A VÀ câu 5 = A
```

---

## Bước 4 — Hiển thị cảnh báo và yêu cầu xác nhận

### Nếu LOW:
```
✓ Risk Level: LOW
Migration này chỉ thay đổi schema và có rollback đầy đủ. Tiếp tục.
```

### Nếu MEDIUM:
```
⚠️  Risk Level: MEDIUM
[Lý do cụ thể dựa trên câu trả lời — vd: "Migration có data changes nhưng có rollback"]

Gõ chính xác câu sau để xác nhận:
  tôi xác nhận đã hiểu rủi ro
```

### Nếu HIGH:
```
🚨 Risk Level: HIGH
[Lý do cụ thể — vd: "Migration INSERT data ảnh hưởng permissions và không có rollback"]

Đây là migration rủi ro cao. Kiểm tra lại:
- [ ] Đã test trên dev chưa?
- [ ] Đã review với teammate chưa?
- [ ] Có plan rollback thủ công chưa?

Gõ chính xác câu sau để xác nhận:
  tôi xác nhận đã hiểu rủi ro
```

**Không tiếp tục bước 5 và 6 cho đến khi nhận được đúng chuỗi `tôi xác nhận đã hiểu rủi ro`.**
Nếu developer gõ sai hoặc gõ tắt, nhắc lại yêu cầu.

---

## Bước 5 — Inject checklist comment vào file migration

Thêm comment block ngay sau dòng `'use strict';` (hoặc đầu file nếu không có):

```js
/**
 * MIGRATION REVIEW CHECKLIST
 * ─────────────────────────────────────────────────────
 * Risk Level : {LOW|MEDIUM|HIGH}
 * Môi trường : {câu trả lời câu 1}
 * Loại       : {Schema only | Data migration | Cả hai}
 * Phạm vi    : {câu trả lời câu 3, hoặc kết quả xác nhận 1/3 từ Bước 1.5, hoặc "N/A" nếu schema only}
 * Permissions: {Có/Không/Không rõ — mô tả ngắn}
 * Rollback   : {Có đầy đủ | Partial | Không}
 *
 * Bảng ảnh hưởng:           ← chỉ điền nếu Câu 2 = B hoặc C
 *   {THAO TÁC} → {ten_bang} ({điều kiện hoặc ước tính phạm vi})
 *   ...
 * Tạo rows mới: {Có | Không | N/A}
 * ─────────────────────────────────────────────────────
 * Xác nhận   : {tên từ git config} – {ngày hôm nay YYYY-MM-DD}
 * ─────────────────────────────────────────────────────
 */
```

---

## Bước 6 — Tạo review doc

Tạo file `docs/migrations/YYYY-MM-DD-{tên-migration-file}-review.md`:

```markdown
# Migration Review: {tên file migration}

**Date:** YYYY-MM-DD
**Risk Level:** {LOW|MEDIUM|HIGH}
**File:** `db/migrations/{tên file}.js`

## Câu hỏi & Trả lời

| # | Câu hỏi | Trả lời |
|---|---------|---------|
| 1 | Môi trường | {trả lời} |
| 2 | Loại migration | {trả lời} |
| 3 | Phạm vi ảnh hưởng | {trả lời hoặc N/A} |
| 4 | Permissions/Visibility | {trả lời hoặc N/A} |
| 5 | Rollback | {trả lời} |

## Phân tích bảng bị ảnh hưởng

*(Chỉ có nếu Câu 2 = B hoặc C)*

| Bảng | Thao tác | Điều kiện | Ước tính phạm vi |
|------|----------|-----------|-----------------|
| {ten_bang} | {INSERT/UPDATE/DELETE/READ} | {WHERE/JOIN condition} | {Toàn bộ / Subset} |

**Phạm vi:** {Toàn bộ | Subset theo điều kiện}  
**Nguồn cập nhật:** {Có lọc nguồn | Không lọc}  
**Insert rows mới:** {Có | Không}  

## Risk Assessment

{Giải thích lý do risk level, các yếu tố đóng góp}

## Xác nhận

Developer đã xác nhận rủi ro: {Có/Không cần — vì LOW}
```

---

## Các trường hợp đặc biệt (luôn áp dụng dù code có vẻ đơn giản)

- `queryInterface.sequelize.query()` raw SQL → luôn hỏi câu 3 và 4
- `down()` chỉ có `return Promise.resolve()` → tự động MEDIUM+
- JOIN sang bảng `user` filter theo `role_id` hoặc permission fields → tự động hỏi câu 4
- File migration có `backfill` trong tên → tự động hỏi câu 3 và 4
