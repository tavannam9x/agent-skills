---
name: migration-review
description: Use BEFORE creating or editing a database migration file (db/migrations/*.js, yarn g:migrate:*, migrate:create) to assess risk, ask clarifying questions, require confirmation for high-risk changes, and document the review.
version: 2.0.0
---

# Migration Review

## Khi nào dùng

Skill này chạy TỰ ĐỘNG (qua hook) mỗi khi Claude tạo hoặc chỉnh sửa file `db/migrations/*.js`.
Cũng có thể gọi thủ công bằng `/migration-review`.

**HARD RULE:** Không được bỏ qua skill này khi làm việc với migration files.

---

## Thứ tự thực hiện

```
Bước 1  → Phân tích tự động file migration
Bước 1.5 → Phân tích bảng bị ảnh hưởng (chỉ khi có data migration)
Bước 2  → Hỏi các câu xác nhận
Bước 2.5 → Điều chỉnh SQL theo câu trả lời (nếu cần)
Bước 3  → Tính Risk Level
Bước 4  → Hiển thị cảnh báo, yêu cầu xác nhận
Bước 5  → Inject checklist comment vào file
Bước 6  → Tạo review doc
```

---

## Bước 1 — Phân tích tự động file migration

Đọc file migration. Xác định các thuộc tính sau:

| Thuộc tính | Cách phát hiện |
|---|---|
| Có data migration? | File chứa INSERT, UPDATE, DELETE, hoặc `sequelize.query` với data-modifying SQL |
| Ảnh hưởng permissions? | JOIN sang bảng `user` + filter theo `role_id`, `is_active`, permission fields |
| Có rollback? | `down()` có nội dung thực sự hay chỉ `return Promise.resolve()` |
| Schema only? | Chỉ có `addColumn`, `removeColumn`, `createTable`, `dropTable`, `addIndex` |
| Có DROP destructive? | `removeColumn`, `dropTable` — destructive dù là schema-only |
| Có INSERT không idempotent? | `INSERT INTO` không có `WHERE NOT EXISTS` hoặc `ON DUPLICATE KEY` |
| Bảng lớn có index? | `addIndex` trên bảng có thể lớn (orders, customers, calls...) |

**Cờ đặc biệt — tự động áp dụng dù code có vẻ đơn giản:**
- `queryInterface.sequelize.query()` raw SQL → luôn hỏi câu 3 và 4
- `down()` chỉ có `return Promise.resolve()` → tự động MEDIUM+
- JOIN sang bảng `user` filter theo `role_id` hoặc permission fields → tự động hỏi câu 4
- File có `backfill` trong tên → tự động hỏi câu 3 và 4
- `removeColumn` hoặc `dropTable` → tự động MEDIUM minimum, hiển thị cảnh báo DROP
- `INSERT INTO` không có `WHERE NOT EXISTS` → cảnh báo idempotency (xem Bước 2.5)

---

## Bước 1.5 — Phân tích bảng bị ảnh hưởng *(chỉ chạy khi có data migration)*

### Cách phân tích

| Thông tin | Cách phát hiện |
|---|---|
| Tên bảng | Tham số đầu tiên của `queryInterface.*()`, tên bảng trong raw SQL |
| Loại thao tác | `INSERT`, `UPDATE`, `DELETE` → data-modifying; `SELECT`, `JOIN` → READ |
| Điều kiện lọc | Mệnh đề `WHERE`, điều kiện `JOIN ON`, tham số `where:` trong Sequelize |
| Ước tính phạm vi | Không có WHERE → "toàn bộ"; có WHERE filter → "subset cụ thể" |

### Present kết quả

```
📊 Phân tích bảng bị ảnh hưởng (tự động):

| Bảng | Thao tác | Điều kiện | Ước tính phạm vi |
|------|----------|-----------|-----------------|
| ten_bang | INSERT | WHERE x = 1 | Subset |
| bang_nguon | READ (JOIN) | JOIN ON ... | Chỉ đọc |
```

### 3 câu xác nhận phạm vi (hỏi từng câu, đợi trả lời)

**Câu xác nhận 1/3 — Phạm vi tổng:**
```
Migration này ảnh hưởng toàn bộ data hay chỉ subset theo điều kiện?
A. Toàn bộ — không có filter đặc biệt
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

### Fallback: hỏi câu 3 gốc nếu

1. SQL dùng tên bảng động (biến, string concatenation, stored procedure).
2. Developer chọn "B. Subset" ở xác nhận 1/3 nhưng điều kiện trong code không rõ enterprise/user cụ thể.

---

## Bước 2 — Hỏi các câu xác nhận

Hỏi **từng câu một**, đợi trả lời. Bỏ qua câu đã rõ từ phân tích code.

**Câu 1 — Môi trường:** Migration sẽ chạy trên môi trường nào?
- A. Chỉ dev/local
- B. Dev + staging
- C. Dev + staging + production

**Câu 2 — Loại migration:** Migration có data changes không?
- A. Schema only (ADD COLUMN, CREATE TABLE, ADD INDEX...)
- B. Có data migration (INSERT/UPDATE/DELETE)
- C. Cả hai

*(Nếu câu 2 = A → chạy Bước 1.5 chưa cần thiết, nhảy sang câu 4)*
*(Nếu câu 2 = B hoặc C → đã chạy Bước 1.5 ở trên rồi)*

**Câu 3 — Phạm vi:** *(chỉ hỏi nếu Bước 1.5 không xác định được rõ)*

Ai/cái gì bị ảnh hưởng bởi data migration này?
- A. Toàn bộ user/data trong hệ thống
- B. Một số enterprise hoặc user cụ thể
- C. Bảng nhỏ, ít dữ liệu, không ảnh hưởng user trực tiếp

**Câu 4 — Permissions/Visibility:** Migration có ảnh hưởng đến quyền truy cập hoặc data visibility không?
- A. Có — user có thể thấy data mới hoặc mất quyền xem
- B. Không rõ
- C. Không

*(Câu 4 KHÔNG bỏ qua dù câu 2 = A nếu migration là DROP COLUMN hoặc DROP TABLE)*

**Câu 5 — Rollback:** `down()` có rollback được không?
- A. Có — rollback đầy đủ
- B. Partial — rollback một phần
- C. Không — `down()` là no-op

---

## Bước 2.5 — Điều chỉnh SQL migration theo câu trả lời

Áp dụng sau khi có đủ câu trả lời, trước khi finalize file migration.

### Idempotency — INSERT rows mới (câu xác nhận 3/3 = A)

| SQL hiện tại | Yêu cầu |
|---|---|
| `INSERT INTO ... SELECT ...` không có guard | **Phải** thêm `WHERE NOT EXISTS (SELECT 1 FROM ten_bang WHERE ...)` hoặc `ON DUPLICATE KEY UPDATE` |
| Đã có guard | OK |

Nếu thiếu → hỏi:
```
INSERT này không có idempotency guard. Nếu migration chạy 2 lần sẽ tạo duplicate data.
Thêm WHERE NOT EXISTS vào không? (Y/N)
```

### Phạm vi (câu xác nhận 1/3 = B)

SQL phải có WHERE / JOIN điều kiện cụ thể thu hẹp phạm vi.

### Câu xác nhận mâu thuẫn với SQL đã có

```
SQL hiện tại dùng [INSERT/UPDATE] nhưng bạn trả lời [câu trả lời].
Cập nhật SQL theo câu trả lời không? (Y/N)
```

---

## Bước 3 — Tính Risk Level

```
HIGH   = (câu 2 = B/C) VÀ (
            câu 3 = A (toàn bộ)
            HOẶC câu 4 = A/B (permissions không rõ)
            HOẶC câu 5 = B/C (không rollback)
         )
         HOẶC (DROP TABLE / DROP COLUMN) VÀ câu 5 = B/C
         HOẶC (câu 1 = C production) VÀ câu 5 = C

MEDIUM = (câu 2 = B/C)
         HOẶC (câu 5 = B/C)
         HOẶC (DROP TABLE / DROP COLUMN) — luôn ít nhất MEDIUM
         HOẶC (INSERT không idempotent chưa được fix)

LOW    = câu 2 = A VÀ câu 5 = A VÀ không có DROP destructive
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
[Lý do cụ thể — vd: "Migration có data changes nhưng có rollback đầy đủ"]

Gõ chính xác câu sau để xác nhận:
  tôi xác nhận đã hiểu rủi ro
```

### Nếu HIGH:
```
🚨 Risk Level: HIGH
[Lý do cụ thể — vd: "Migration INSERT data ảnh hưởng toàn bộ user và không có rollback"]

Đây là migration rủi ro cao. Kiểm tra lại:
- [ ] Đã test trên dev/staging chưa?
- [ ] Đã review với teammate chưa?
- [ ] Có plan rollback thủ công chưa?
- [ ] Nếu lên production: đã backup DB chưa?

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
 * Phạm vi    : {kết quả xác nhận 1/3 hoặc câu 3, hoặc "N/A" nếu schema only}
 * Permissions: {Có/Không/Không rõ — mô tả ngắn}
 * Rollback   : {Có đầy đủ | Partial | Không}
 * Idempotent : {Có | Không áp dụng}    ← chỉ điền nếu có INSERT
 *
 * Bảng ảnh hưởng:                      ← chỉ điền nếu có data migration
 *   {THAO TÁC} → {ten_bang} ({điều kiện hoặc ước tính phạm vi})
 * Tạo rows mới: {Có | Không | N/A}
 * ─────────────────────────────────────────────────────
 * Xác nhận   : {git config user.name, fallback "unknown"} – {YYYY-MM-DD}
 * ─────────────────────────────────────────────────────
 */
```

---

## Bước 6 — Tạo review doc

Tạo file `docs/migrations/YYYY-MM-DD-{tên-migration-file}-review.md`.
Nếu folder `docs/migrations/` chưa tồn tại, tạo mới.

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
| {ten_bang} | {INSERT/UPDATE/DELETE/READ} | {condition} | {Toàn bộ / Subset} |

**Phạm vi:** {Toàn bộ | Subset theo điều kiện}
**Nguồn cập nhật:** {Có lọc nguồn | Không lọc}
**Insert rows mới:** {Có | Không}
**Idempotent:** {Có (WHERE NOT EXISTS) | Không áp dụng}

## Risk Assessment

{Giải thích lý do risk level, các yếu tố đóng góp}

## Xác nhận

Developer đã xác nhận rủi ro: {Có / Không cần — vì LOW}
```
