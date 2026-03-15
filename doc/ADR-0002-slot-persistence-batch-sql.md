# ADR-0002: Batch SQL Operations in Slot Persistence

**Status:** Proposed (revised per architect review)
**Date:** 2026-03-15
**Context:** Account slot save is the dominant bottleneck during OFX import with a PostgreSQL backend
**Decision:** Option A
**Depends on:** ADR-0001 (for batch transaction wrapping; independent implementation)

## Problem

When an account with a large Bayesian import map is committed to a PostgreSQL backend over a network, the commit takes ~33 seconds. This is Root Cause 1 from ADR-0001, which was explicitly deferred.

Observed: an account with 1,094 slots (accumulated from years of OFX imports into a credit card account) takes ~33 seconds to commit over a network PostgreSQL connection with `synchronous_commit = on`.

The bottleneck is `gnc_sql_slots_save` (`gnc-slots-sql.cpp:634`), which uses a **delete-all-then-reinsert-all** strategy with **one SQL statement per round-trip**.

## Root Cause

### Slot Table Structure

The `slots` table stores the flattened KVP (Key-Value Pair) tree for every QofInstance. Each row is one KVP leaf or intermediate FRAME/GLIST node:

| Column | Type | Notes |
|---|---|---|
| `id` | INT | Primary key, auto-increment |
| `obj_guid` | GUID | The owning object's GUID (or a synthetic GUID for nested FRAMEs) |
| `name` | STRING | The slot path segment |
| `slot_type` | INT | KvpValue type enum |
| `int64_val`, `string_val`, `double_val`, `timespec_val`, `guid_val`, `numeric_val`, `gdate_val` | various | Only one is populated per row, determined by `slot_type` |

There is an index on `obj_guid` but **no unique constraint** on `(obj_guid, name)`.

### KVP Tree → Slot Rows

The Bayesian import map for an account is stored as a nested KVP tree:

```
imap/
  bayes/
    GROCERY STORE/           ← FRAME
      <dest_account_guid>    ← INT64 (token count)
    PHARMACY 1234/           ← FRAME
      <dest_account_guid>    ← INT64 (token count)
    ... (~500 tokens)
```

Each FRAME node generates a synthetic GUID. Its children are stored with `obj_guid` = that synthetic GUID. For an account with ~500 Bayesian tokens averaging 1 destination account each, the slot table contains:

- ~3 ancestor FRAMEs (`imap`, `bayes`, and a few other top-level slots)
- ~500 FRAME-type rows (one per token), each with a `guid_val` pointing to a synthetic GUID
- ~500 INT64-type leaf rows (the token counts)
- Plus non-Bayesian slots (preferences, online_id, etc.)
- **Total: ~1,094 rows**

### DELETE Path: Recursive Round-Trips

`gnc_sql_slots_delete` (`gnc-slots-sql.cpp:659`) works recursively:

```
gnc_sql_slots_delete(account_guid):
    SELECT * FROM slots
        WHERE obj_guid = account_guid
        AND slot_type IN (FRAME, GLIST)
        AND guid_val IS NOT NULL              -- 1 round-trip (SELECT)
    for each child_guid in results:
        gnc_sql_slots_delete(child_guid)      -- RECURSE
    DELETE FROM slots WHERE obj_guid = account_guid  -- 1 round-trip (DELETE)
```

For the Bayesian map structure with depth D and F FRAME nodes, this produces:
- **F SELECT round-trips** (one per FRAME node, most returning 0-1 children)
- **F DELETE round-trips** (one per unique `obj_guid`)

With F ≈ 503 (3 ancestors + 500 token FRAMEs): **~1,006 round-trips** just for deletion.

### INSERT Path: Per-Row Round-Trips

`save_slot` (`gnc-slots-sql.cpp:565`) is called once per KVP node via `for_each_slot_temp`. Each call executes `do_db_operation(OP_DB_INSERT, ...)`, which builds and executes a single-row INSERT statement:

```sql
INSERT INTO slots (obj_guid, name, slot_type, int64_val, string_val, ...)
VALUES ('guid', 'path', 1, 42, NULL, ...);
```

For N = 1,094 slots: **1,094 INSERT round-trips**.

### Total

| Phase | Round-trips | At 30ms each |
|---|---|---|
| DELETE (recursive SELECT + DELETE) | ~1,006 | ~30s |
| INSERT (per-row) | ~1,094 | ~33s |
| **Total** | **~2,100** | **~63s** |

The observed 33 seconds suggests the DELETE is faster than the worst case (many leaf FRAMEs return empty SELECTs quickly), or the per-round-trip latency is ~15ms. Either way, the fix must address both paths.

## Decision: Batch SQL Operations in gnc_sql_slots_save (Option A)

Replace the per-row SQL operations in both the DELETE and INSERT paths with batched equivalents that reduce round-trips from O(N) to O(N/batch_size + depth).

### Batch DELETE

Replace recursive `gnc_sql_slots_delete` with a level-by-level collection strategy:

1. **Collect GUIDs level by level.** Starting from the root `obj_guid`, SELECT all FRAME/GLIST child `guid_val`s. Then SELECT all FRAME/GLIST child `guid_val`s for those GUIDs (using `IN (...)`). Repeat until no more children are found.

2. **Delete in one statement.** Issue a single `DELETE FROM slots WHERE obj_guid IN (root_guid, child1, child2, ...)`.

```
Level 0:  SELECT guid_val FROM slots
              WHERE obj_guid = '<account_guid>'
              AND slot_type IN (FRAME, GLIST)
              AND guid_val IS NOT NULL           -- 1 round-trip → {G1}
Level 1:  SELECT guid_val FROM slots
              WHERE obj_guid IN ('G1')
              AND slot_type IN (FRAME, GLIST)
              AND guid_val IS NOT NULL           -- 1 round-trip → {G2}
Level 2:  SELECT guid_val FROM slots
              WHERE obj_guid IN ('G2')
              AND slot_type IN (FRAME, GLIST)
              AND guid_val IS NOT NULL           -- 1 round-trip → {G3..G502}
Level 3:  SELECT guid_val FROM slots
              WHERE obj_guid IN ('G3', ..., 'G502')
              AND slot_type IN (FRAME, GLIST)
              AND guid_val IS NOT NULL           -- 1 round-trip → {} (leaves)
Final:    DELETE FROM slots
              WHERE obj_guid IN ('<account_guid>', 'G1', 'G2', 'G3', ..., 'G502')
                                                  -- 1 round-trip
```

**Total: depth + 1 round-trips.** For the Bayesian map (depth 4): **5 round-trips** instead of ~1,006.

**GUID extraction:** The level-by-level SELECTs return `guid_val` as strings via `GncSqlRow::get_string_at_col()` (returning `std::optional<std::string>`), then convert to `GncGUID` via `string_to_guid()`. This is the same extraction mechanism used by the existing recursive `gnc_sql_slots_delete` (line 671-687 of `gnc-slots-sql.cpp`), ensuring consistent handling of database-specific GUID representations across all backends.

The `IN` clause at level 3 contains ~500 GUIDs. At 32 hex chars + quotes + comma per GUID, that's ~17KB — well within SQL statement size limits for all supported databases (PostgreSQL: effectively unlimited; MySQL: `max_allowed_packet` default 4MB; SQLite: `SQLITE_LIMIT_SQL_LENGTH` default 1MB).

Both SELECT and DELETE `IN` clauses are subject to a single threshold constant, `GUID_IN_CLAUSE_LIMIT = 500`. If a level's `IN` clause would exceed this limit, split the SELECT into multiple queries of up to `GUID_IN_CLAUSE_LIMIT` GUIDs each and union the results. The final DELETE splits the same way: if `all_guids` exceeds `GUID_IN_CLAUSE_LIMIT`, issue multiple independent `DELETE FROM slots WHERE obj_guid IN (...)` statements. These DELETE chunks have no ordering requirements — no FK constraints — and can be issued in any order. For the typical case (503 GUIDs for the Bayesian map), this means 2 DELETE statements.

### Batch INSERT

Replace per-row `do_db_operation(OP_DB_INSERT)` calls with an accumulate-then-flush approach that reuses the existing `col_table` metadata for column definitions.

1. **Accumulate.** Walk the KVP tree as before using `for_each_slot_temp`. For each slot (including recursive FRAME/GLIST children), capture the populated `slot_info_t` state into a `std::vector<slot_info_t>` (the accumulator).

2. **Flush in batches.** After the tree walk, call a new `GncSqlBackend::do_db_operation_batch` method that accepts the `col_table` and the vector of objects, and generates multi-row INSERT statements:

```sql
INSERT INTO slots (obj_guid, name, slot_type, int64_val, string_val,
                   double_val, timespec_val, guid_val,
                   numeric_val_num, numeric_val_denom, gdate_val)
VALUES ('guid1', 'path1', 1, 42, NULL, NULL, NULL, NULL, NULL, NULL, NULL),
       ('guid2', 'path2', 4, NULL, NULL, NULL, NULL, 'child_guid', NULL, NULL, NULL),
       ('guid3', 'path3', 1, 7, NULL, NULL, NULL, NULL, NULL, NULL, NULL),
       ...;  -- up to 100 rows per statement
```

The column list and per-row value extraction are derived from the same `col_table` (`EntryVec`) used by the existing single-row `do_db_operation(OP_DB_INSERT)` path, via `get_object_values()`. This keeps the column definition in one place — if a column is added to the slots table, both paths pick it up automatically. The `id` column (flagged `COL_AUTOINC`) is skipped by the existing `is_autoincr()` check in `get_object_values()`, which is correct for multi-row INSERT (auto-increment behavior is well-defined for multi-row INSERTs across all three backends: PostgreSQL uses sequences, MySQL uses `AUTO_INCREMENT`, SQLite uses `ROWID`).

**Total: ceil(N / 100) round-trips.** For N = 1,094: **11 round-trips** instead of 1,094.

The batch size of 100 is conservative. Each row is ~200 bytes of SQL, so a batch of 100 is ~20KB. This is well within limits for all supported databases. The batch size can be tuned upward if profiling justifies it.

### GUID Generation

FRAME and GLIST slot types generate synthetic GUIDs during the tree walk. The current `save_slot` generates these GUIDs and immediately INSERTs the parent row (so the GUID exists in the table before children reference it). With the batched approach, GUIDs are generated during accumulation but all rows are INSERTed together in the flush step. This is correct because:

1. Child rows reference the synthetic GUID via `obj_guid`, not via a foreign key constraint. There are no FK constraints on the slots table.
2. All rows within a multi-row INSERT are visible to the database at the same time.
3. Even across batch boundaries (parent in batch 1, children in batch 2), the rows are all within the same database transaction (either the batch from ADR-0001 or the per-object transaction from `GncSqlBackend::commit`).

**Transaction boundary with ADR-0001:** If ADR-0001 wraps multiple account commits in a single transaction, and one account's `do_db_operation_batch` (or `gnc_sql_slots_delete_batch`) fails, the ADR-0001 transaction rolls back *all* accounts in the batch. This is correct for atomicity — either all accounts in the batch are saved or none are. This is a behavioral change from the current per-object failure isolation, where one account's slot save failure does not affect other accounts. The change is intentional: partial saves of a batch are a worse outcome than a clean rollback.

### SQL Value Escaping

String values in the VALUES clauses must be properly escaped. Since `do_db_operation_batch` reuses `get_object_values()` → `add_to_query()` — the same path as `build_insert_statement` for single-row INSERTs — escaping is handled by the existing column table entry implementations. Specifically, `GncSqlColumnTableEntryImpl<CT_STRING>::add_to_query` calls `sql_be->quote_string()`, which delegates to `GncDbiSqlConnection::quote_string()` (`dbi_conn_quote_string_copy`). This provides database-specific escaping for all string types.

GUID values are hex strings extracted via `guid_to_string()` and require no special escaping beyond quoting.

### Changes

#### `gnc-slots-sql.cpp` — Batch DELETE

Replace `gnc_sql_slots_delete` with a new function `gnc_sql_slots_delete_batch`:

```cpp
static gboolean
gnc_sql_slots_delete_batch (GncSqlBackend* sql_be, const GncGUID* guid);
```

Implementation:
1. Stringify the root GUID. Add it to a `std::vector<std::string> all_guids`.
2. Initialize `current_level` = `{root_guid_string}`.
3. Loop while `current_level` is non-empty:
   - Build `SELECT guid_val FROM slots WHERE obj_guid IN (<current_level>) AND slot_type IN (FRAME, GLIST) AND guid_val IS NOT NULL`.
   - Execute and collect results into `next_level`.
   - Append `next_level` to `all_guids`.
   - Set `current_level = next_level`.
4. Build `DELETE FROM slots WHERE obj_guid IN (<all_guids>)`.
5. Execute and return result.

If `all_guids` exceeds `GUID_IN_CLAUSE_LIMIT` (500), split into multiple `DELETE FROM slots WHERE obj_guid IN (...)` statements. No ordering constraints — each chunk is independent (no FK constraints). Similarly, if a level's `current_level` exceeds the limit, split the SELECT at that level into multiple queries and union the results.

The existing `gnc_sql_slots_delete` function signature and callers are unchanged. Only the internal implementation changes.

#### `gnc-sql-backend.hpp` / `gnc-sql-backend.cpp` — `do_db_operation_batch`

Add a new method to `GncSqlBackend`:

```cpp
bool do_db_operation_batch (const char* table_name,
                            QofIdTypeConst obj_name,
                            const std::vector<gpointer>& objects,
                            const EntryVec& table,
                            size_t batch_size = 100) noexcept(false);
```

Implementation:
1. Build the column list string once from `table`, skipping `is_autoincr()` columns (same filter as `get_object_values()`).
2. For each object in `objects`, call `get_object_values()` to extract the column values — this reuses the same `col_table` metadata and getter functions as the single-row path.
3. Format each object's values as a parenthesized tuple.
4. Concatenate up to `batch_size` tuples into a multi-row INSERT statement and execute via `execute_nonselect_statement()`.
5. Return false on the first failure.

This keeps column definitions in one place (`col_table`). If a column is ever added to the slots table, both the single-row and batch paths pick it up automatically.

**Design notes:**
- **Not `const`:** This method executes INSERT statements, mutating database and connection state. Consistent with the existing `do_db_operation`, which is also non-const.
- **Not `noexcept`:** SQL execution can throw on connection failures or allocation errors in libdbi. If `noexcept` were specified, any throw would call `std::terminate()`. The existing `do_db_operation` uses `noexcept` but catches internally; for `do_db_operation_batch` we let exceptions propagate since the caller (`gnc_sql_slots_save`) already handles failure via `is_ok`.
- **`gpointer` type safety:** The `objects` parameter is `std::vector<gpointer>` (void-pointer), consistent with the existing `do_db_operation` signature. This is intentional for generality — `do_db_operation_batch` is designed to be usable for any table type (transactions, splits, etc.), not just slots. Callers are responsible for ensuring type consistency between the `objects` vector and the `col_table` getter functions. A template wrapper could add compile-time safety but is out of scope for this ADR.
- **Generality:** This method is a general-purpose addition to `GncSqlBackend`, usable by any object backend that needs multi-row INSERT. The slots use case is the first consumer; other backends (e.g., bulk transaction import) can adopt it later without further API changes.

#### `gnc-slots-sql.cpp` — Batch INSERT

The accumulator is a `std::vector<slot_info_t>` owned by `gnc_sql_slots_save` and passed to `save_slot` as a separate parameter. `slot_info_t` is **not** modified — no accumulator pointer is added to the struct. This avoids turning a data-carrying struct into a dispatch mechanism.

Since `save_slot` is a static function with exactly one call site (`gnc_sql_slots_save`), the `do_db_operation` calls are replaced unconditionally. The signature of `save_slot` changes to accept the accumulator:

```cpp
static void
save_slot (const char* key, KvpValue* value, slot_info_t & slot_info,
           std::vector<slot_info_t>& accum);
```

The `for_each_slot_temp` callback doesn't match this signature directly. Wrap `save_slot` in a lambda at the call site in `gnc_sql_slots_save`:

```cpp
auto save_cb = [&accum](const char* key, KvpValue* value,
                        slot_info_t& info) {
    save_slot(key, value, info, accum);
};
pFrame->for_each_slot_temp(save_cb, slot_info);
```

**Copy semantics for accumulated `slot_info_t`:** The tree walk mutates `slot_info` as it progresses, and the FRAME/GLIST cases create temporary `KvpValue` objects (wrapping synthetic GUIDs) that are `delete`d after the current iteration (lines 594, 616). A shallow copy of `slot_info_t` into the accumulator would leave dangling pointers. The fields require the following treatment:

| Field | Copy strategy | Rationale |
|---|---|---|
| `be` (GncSqlBackend*) | Shallow — safe | Stable for the entire save lifetime |
| `guid` (const GncGUID*) | **Deep copy required** | For FRAME/GLIST child rows, points to a synthetic GUID owned by a temporary KvpValue that is deleted after the current `save_slot` call. The accumulated copy must own its own `GncGUID`. |
| `is_ok` (gboolean) | Value — safe | Primitive |
| `pKvpValue` (KvpValue*) | **Deep copy required** | For FRAME/GLIST parent rows, `save_slot` sets this to `new KvpValue{guid}` then deletes it on lines 594/616. The accumulated copy must own its own `KvpValue`. For leaf rows, `pKvpValue` points into the in-memory KVP tree, which is stable for the save lifetime — but deep-copying unconditionally is simpler and safe. |
| `path` (std::string) | Value — safe | `std::string` copy constructor handles this |
| `parent_path` (std::string) | Value — safe | Same |
| `pKvpFrame` (KvpFrame*) | Shallow — safe | Points into the in-memory KVP tree, stable for save lifetime. Not read by `get_object_values()`. |
| `value_type` (KvpValue::Type) | Value — safe | Enum, copied by value |
| `pList` (GList*) | Shallow — safe | Not read by the INSERT path's getter functions |
| `context` (context_t) | Value — safe | Enum, copied by value |

Implement this as a static helper `snapshot_for_insert` that produces a deep-copied `slot_info_t` with owned `GncGUID` and `KvpValue`:

```cpp
static slot_info_t
snapshot_for_insert (const slot_info_t& src)
{
    slot_info_t snap = src;                       // shallow copy (strings copy by value)
    snap.guid = new GncGUID(*src.guid);           // deep copy GUID
    snap.pKvpValue = new KvpValue(*src.pKvpValue); // deep copy KvpValue
    return snap;
}
```

The accumulated `slot_info_t` objects own their `guid` and `pKvpValue` allocations. They must be cleaned up after `do_db_operation_batch` returns. Since the vector is local to `gnc_sql_slots_save`, add a cleanup loop:

```cpp
for (auto& s : accum)
{
    delete s.guid;
    delete s.pKvpValue;
}
```

Modify `save_slot` — replace the `do_db_operation` calls:

```cpp
// Where currently:
slot_info.is_ok = slot_info.be->do_db_operation(OP_DB_INSERT, TABLE_NAME,
                                                TABLE_NAME, &slot_info,
                                                col_table);
// Change to:
accum.push_back(snapshot_for_insert(slot_info));
```

This applies to all three cases in `save_slot` (default, FRAME, GLIST).

Modify `gnc_sql_slots_save`:

```cpp
gboolean
gnc_sql_slots_save (GncSqlBackend* sql_be, const GncGUID* guid,
                    gboolean is_infant, QofInstance* inst)
{
    // ... existing guards ...

    if (!sql_be->pristine() && !is_infant)
        gnc_sql_slots_delete_batch (sql_be, guid);  // was: gnc_sql_slots_delete

    std::vector<slot_info_t> accum;
    slot_info.be = sql_be;
    slot_info.guid = guid;
    auto save_cb = [&accum](const char* key, KvpValue* value,
                            slot_info_t& info) {
        save_slot(key, value, info, accum);
    };
    pFrame->for_each_slot_temp (save_cb, slot_info);

    if (slot_info.is_ok && !accum.empty())
    {
        std::vector<gpointer> ptrs;
        ptrs.reserve(accum.size());
        for (auto& s : accum) ptrs.push_back(&s);
        slot_info.is_ok = sql_be->do_db_operation_batch(
            TABLE_NAME, TABLE_NAME, ptrs, col_table);
    }

    for (auto& s : accum)  // cleanup deep-copied fields
    {
        delete s.guid;
        delete s.pKvpValue;
    }

    return slot_info.is_ok;
}
```

#### No changes to `gnc-slots-sql.h`

The public API (`gnc_sql_slots_save`, `gnc_sql_slots_delete`, `gnc_sql_slots_load`) is unchanged. The optimization is purely internal.

### Error Handling

**Multi-row INSERT failure:** If a batch INSERT fails (e.g., constraint violation, connection error), `do_db_operation_batch` returns false. `gnc_sql_slots_save` propagates the failure to its caller (`GncSqlBackend::commit`), which rolls back the per-object savepoint. This is identical to the current behavior when a single-row INSERT fails — the same PERR logging and rollback path applies.

**Batch DELETE failure:** If the DELETE statement fails, `gnc_sql_slots_delete_batch` returns false. `gnc_sql_slots_save` does not currently check the DELETE return value (line 648: `(void)gnc_sql_slots_delete(...)`). This pre-existing issue is preserved. A follow-up could add error checking, but that is not part of this ADR's scope.

**Partial batch failure:** If the 5th batch INSERT succeeds but the 6th fails, 500 rows are inserted and ~594 are not. The per-object savepoint (from `GncSqlBackend::commit`) rolls back all of them — the entire slot save is atomic within the savepoint. If the ADR-0001 batch transaction is active, it's atomic within the outer transaction.

### Runtime Behavior

Before:
```sql
-- DELETE phase (~1,006 round-trips)
SELECT * FROM slots WHERE obj_guid='acct' AND slot_type IN (4,5)...;
SELECT * FROM slots WHERE obj_guid='G1' AND slot_type IN (4,5)...;
DELETE FROM slots WHERE obj_guid='G1';
SELECT * FROM slots WHERE obj_guid='G2' AND slot_type IN (4,5)...;
DELETE FROM slots WHERE obj_guid='G2';
... -- ~500 more SELECT+DELETE pairs
DELETE FROM slots WHERE obj_guid='acct';

-- INSERT phase (~1,094 round-trips)
INSERT INTO slots (...) VALUES (...);   -- row 1
INSERT INTO slots (...) VALUES (...);   -- row 2
...                                     -- × 1,094
```

After:
```sql
-- DELETE phase (5 round-trips for depth-4 tree)
SELECT guid_val FROM slots WHERE obj_guid IN ('acct') AND slot_type IN (4,5)...;
SELECT guid_val FROM slots WHERE obj_guid IN ('G1') AND slot_type IN (4,5)...;
SELECT guid_val FROM slots WHERE obj_guid IN ('G2') AND slot_type IN (4,5)...;
SELECT guid_val FROM slots WHERE obj_guid IN ('G3',...,'G502') AND slot_type IN (4,5)...;
DELETE FROM slots WHERE obj_guid IN ('acct','G1','G2','G3',...,'G502');

-- INSERT phase (11 round-trips for 1,094 rows)
INSERT INTO slots (...) VALUES (...),(...),(...), ... ;  -- rows 1-100
INSERT INTO slots (...) VALUES (...),(...),(...), ... ;  -- rows 101-200
...                                                      -- × 11
```

### Assumptions

- **No foreign key constraints on the slots table.** The `obj_guid` column references synthetic GUIDs for nested FRAME/GLIST structures, but there is no FK constraint. This means rows can be inserted in any order and deleted in bulk without cascading concerns. Verified by inspecting the table creation code (`gnc-slots-sql.cpp:846`).

- **Multi-row INSERT is supported by all target databases.** PostgreSQL (all versions), MySQL 5.x+, and SQLite 3.7.11+ all support `INSERT INTO ... VALUES (...), (...), ...`. GnuCash's minimum dependency versions all satisfy this.

- **Statement size limits are not exceeded.** With a batch size of 100 rows at ~200 bytes each, the largest INSERT statement is ~20KB. The largest DELETE `IN` clause is ~17KB (500 GUIDs × 34 bytes each). These are well within the limits of all target databases.

- **The `is_infant` fast path is preserved.** For newly created objects (`is_infant = TRUE`), the DELETE phase is skipped entirely (existing behavior at line 646). The INSERT phase uses the batched path. Since infant objects have no existing slots, this is strictly faster.

- **`sync()` uses `gnc_sql_slots_save` for every object.** The full book save (`GncSqlBackend::sync`) iterates all objects and saves their slots. The batched approach benefits `sync()` as well — a book with 50 accounts averaging 200 slots each would go from ~10,000 INSERT round-trips to ~100.

- **Concurrent access race condition (pre-existing, window shape changes).** If two GnuCash instances save the same account concurrently (possible with network backends), the delete-all/reinsert-all strategy has a race: instance A deletes, instance B deletes (no-op), instance A inserts, instance B inserts (duplicates). This is a pre-existing bug not introduced by this ADR. However, the batched DELETE changes the race window shape: the current recursive DELETE immediately deletes each GUID's rows as they're discovered, while the batched DELETE collects all GUIDs across multiple SELECTs before issuing any DELETE. A concurrent writer could insert new child GUIDs between the level-N SELECT and the final DELETE — those children would be orphaned (their `obj_guid` references a synthetic GUID that no longer has a parent row). This is the same class of bug (concurrent write during delete-reinsert) but with a slightly wider collection window. Fixing it would require row-level locking or advisory locks, which is out of scope.

### Success Criteria

Measured against the user's environment: PostgreSQL over network, `synchronous_commit = on`, account with ~1,094 slots.

| Benchmark | Before | After (target) |
|---|---|---|
| Account commit (1,094 slots, network) | ~33s | < 1s |
| Account commit (1,094 slots, localhost) | ~2s | < 0.2s |
| Full book save (`sync()`) with 50 accounts, ~10,000 total slots | ~5 min | < 10s |

Note: the network PostgreSQL target of "< 1s" is achievable but tight. With 16 round-trips at 30ms each, SQL alone is ~480ms. Add the KVP tree walk and GUID extraction and realistic wall-clock time is ~600-700ms. Do not promise sub-500ms externally.

If the localhost improvement is small, the bottleneck is per-statement parsing overhead rather than round-trip latency, and prepared statements should be investigated.

### Test Plan

1. **Unit test: `do_db_operation_batch` round-trip** (`libgnucash/backend/sql/test/` or `libgnucash/backend/dbi/test/`)
   - Create a SQLite test database. Call `do_db_operation_batch` with 250 slot_info_t objects and batch size 100. Verify 250 rows are inserted. Verify 3 INSERT statements were executed (100 + 100 + 50).
   - Call with 0 objects. Verify no statements executed.

2. **Unit test: `gnc_sql_slots_delete_batch` correctness** (`libgnucash/backend/dbi/test/`)
   - Populate a slots table with a 3-level deep FRAME tree (root → 5 FRAMEs → 3 leaves each = 21 rows).
   - Call `gnc_sql_slots_delete_batch`. Verify all 21 rows are deleted.
   - Populate a slots table with a flat structure (no FRAMEs). Call `gnc_sql_slots_delete_batch`. Verify all rows are deleted.

3. **Integration test: save/load round-trip** (`libgnucash/backend/dbi/test/`)
   - Create an account with ~100 Bayesian import map entries. Save via `gnc_sql_slots_save` (batched). Load via `gnc_sql_slots_load`. Verify the loaded KVP tree is identical to the saved one.
   - Same test with GLIST-type slots.

4. **Regression test: infant object fast path**
   - Create a new account (is_infant = TRUE). Save slots. Verify no DELETE is executed. Verify slots are saved correctly via batched INSERT.

5. **Regression test: `sync()` full book save**
   - Save a book with multiple accounts to SQLite. Load it back. Verify all data is intact. This exercises the batched path for every object type that has slots.

6. **Unit test: SQL-hostile string values** (`libgnucash/backend/dbi/test/`)
   - Create slots with string values containing: single quotes (`O'Brien`), backslashes (`C:\path`), NUL bytes, Unicode (`Café`), SQL injection attempts (`'; DROP TABLE slots;--`), empty strings (`""`), and strings at `SLOT_MAX_STRINGVAL_LEN` (4096 bytes).
   - Save via batched INSERT. Load back. Verify all values round-trip exactly. This exercises `quote_string()` through the `get_object_values()` → `add_to_query()` path.
   - Bayesian tokens come from OFX transaction descriptions (bank-controlled but user-visible data), so this is a real input surface.

7. **Unit test: empty KVP tree** (`libgnucash/backend/dbi/test/`)
   - Create an account with zero slots (e.g., a freshly imported account before Bayesian learning).
   - Call `gnc_sql_slots_save`. Verify `for_each_slot_temp` produces nothing, `do_db_operation_batch` is not called, and the DELETE still runs (unless `is_infant`).
   - This is the common case for newly imported accounts.

8. **Unit test: partial batch failure rollback** (`libgnucash/backend/dbi/test/`)
   - Create an account with ~250 slots. Inject a failure after the 2nd batch INSERT (e.g., by closing the database connection or inserting a constraint-violating row into the 3rd batch via a test hook).
   - Verify that `gnc_sql_slots_save` returns false.
   - Verify that the slots table is unchanged from its pre-save state (the per-object savepoint rolled back all batches, including the successful ones).
   - This is the most important correctness property of the batched approach: partial INSERT must not leave the database in an inconsistent state.

9. **Performance benchmark** (manual)
   - PostgreSQL over network, account with ~1,094 slots.
   - Measure wall-clock time of account commit before and after.
   - Compare against success criteria.

### Characteristics

- **Effort:** ~100 lines for `do_db_operation_batch` in `gnc-sql-backend.cpp` + ~80 lines for `gnc_sql_slots_delete_batch` (including IN-clause splitting and GUID extraction) + ~50 lines of modifications to `save_slot` and `gnc_sql_slots_save` + ~20 lines for error propagation = **~250 lines of new/modified code** + ~250 lines of tests (including escaping and empty-tree edge cases).
- **Risk:** Low-moderate. The public API is unchanged. The KVP tree walk is unchanged. Only the SQL generation and execution are modified. There is no legacy fallback path — `save_slot` is static with exactly one call site (`gnc_sql_slots_save`), so the per-row `do_db_operation` calls are replaced unconditionally.

## Rejected: Incremental Slot Updates (Option B)

Instead of delete-all/reinsert-all, diff the in-memory KVP tree against the database and apply only the changes (INSERT new slots, DELETE removed slots, UPDATE modified slots).

### Why Rejected

1. **FRAME GUIDs are regenerated on every save.** The current `save_slot` generates new synthetic GUIDs for every FRAME and GLIST node on every save. The database contains the *previous* set of synthetic GUIDs. There is no stable identifier to match an in-memory FRAME to its database counterpart — the slot `name` is the path segment, but the `obj_guid` (which links children to parents) changes every save. A diff-based approach would need to either (a) stabilize the synthetic GUIDs across saves (invasive change to the KVP persistence model) or (b) reconstruct the parent-child relationships by path, which is error-prone for nested FRAME/GLIST structures.

2. **No dirty tracking in the KVP framework.** `KvpFrame` is a plain `std::map` with no change list or dirty flags. `QofInstance` has a single `dirty` flag for the entire object. To compute a diff, the implementation would need to either load the database state before each save (adding a SELECT-all round-trip) or maintain a shadow copy of the last-saved KVP tree in memory (doubling memory usage for KVP data). Neither is free.

3. **Marginal benefit over Option A.** Option A reduces round-trips from ~2,100 to ~16 (a 130x improvement). Option B would reduce them further to ~5-10 (the diff itself), but the absolute time difference between 16 and 5 round-trips at 30ms each is 330ms vs 150ms. Not worth the complexity and risk.

4. **Risk of state divergence.** If the diff logic has a bug — a slot that should be deleted is kept, or a slot that should be updated is skipped — the database state silently diverges from the in-memory state. This is a data integrity risk in a financial application. The delete-all/reinsert-all strategy is crude but correct by construction: the database always ends up matching the in-memory state.

Option B should not be built unless profiling after Option A demonstrates that slot persistence is still a bottleneck. If that day comes, the FRAME GUID instability problem (point 1) must be solved first.

## Rejected: Schema Change for UPSERT (Option C)

Add a `UNIQUE(obj_guid, name)` constraint to the slots table, enabling `INSERT ... ON CONFLICT DO UPDATE` (PostgreSQL), `INSERT OR REPLACE` (SQLite), or `INSERT ... ON DUPLICATE KEY UPDATE` (MySQL).

### Why Rejected

1. **Requires a schema migration.** Adding a unique constraint to a table with potentially millions of rows is a heavyweight operation that locks the table. GnuCash has a version-based table upgrade mechanism (`gnc-slots-sql.cpp:855-887`), but adding a unique constraint requires verifying that no duplicates exist first. The current delete-all/reinsert-all strategy can produce transient duplicates if a save is interrupted.

2. **Does not reduce round-trips.** UPSERT still executes one statement per row. It eliminates the DELETE step but the INSERT round-trip count is unchanged. Option A is strictly better.

3. **Database-specific syntax.** Each database has a different UPSERT syntax. The abstraction layer (`do_db_operation`) does not support UPSERT, so database-specific code paths would be needed.

## Future Work

1. **Incremental slot updates** (Option B above). If profiling after Option A shows that the 16 round-trips are still a bottleneck (unlikely for reasonable slot counts), a diff-based approach can be explored. The prerequisite is stabilizing FRAME synthetic GUIDs across saves.

2. **Prepared statements.** If per-statement parsing overhead is significant (visible on localhost benchmarks), libdbi's `dbi_conn_prepare` / `dbi_conn_pquery` could be used to prepare the INSERT template once and execute it N times. This reduces parsing overhead but not round-trip count.

3. **Slot count monitoring.** Accounts that accumulate thousands of Bayesian import map entries over years will continue to grow. A periodic cleanup or compaction mechanism (e.g., pruning low-count tokens) would reduce the slot count and keep commit times bounded regardless of optimization.
