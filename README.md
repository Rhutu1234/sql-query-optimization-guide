# Query Optimization

*A deep-dive walkthrough of SQL query optimization as a practical, systematic discipline — covering SARGability (why a query's own shape can disqualify an index before the optimizer ever gets a chance to use it), function-wrapped columns and implicit conversions as the two most common SARGability killers, filtering as early and as precisely as possible, avoiding `SELECT *`, set-based thinking versus row-by-row processing, query rewriting techniques, and a unified, end-to-end workflow that draws together this series' SQL Indexes, Execution Plans, Joins, Transactions, Deadlocks, and Isolation Levels guides into one practical process for making a slow query fast.*

---

## Table of Contents

1. [Introduction](#introduction)
2. [SARGability: The Single Most Important Concept in This Guide](#1-sargability-the-single-most-important-concept-in-this-guide)
3. [Function-Wrapped Columns: The Most Common SARGability Killer](#2-function-wrapped-columns-the-most-common-sargability-killer)
4. [Implicit Conversions: The Silent SARGability Killer](#3-implicit-conversions-the-silent-sargability-killer)
5. [Leading Wildcards and LIKE](#4-leading-wildcards-and-like)
6. [SELECT *: Why It Costs More Than It Looks Like](#5-select--why-it-costs-more-than-it-looks-like)
7. [Filtering Early and Precisely](#6-filtering-early-and-precisely)
8. [Set-Based Thinking vs. Row-by-Row Processing](#7-set-based-thinking-vs-row-by-row-processing)
9. [OR Conditions and UNION ALL Rewrites](#8-or-conditions-and-union-all-rewrites)
10. [Pagination: OFFSET/FETCH and Keyset Pagination](#9-pagination-offsetfetch-and-keyset-pagination)
11. [Parameterization and Plan Reuse](#10-parameterization-and-plan-reuse)
12. [Query Rewriting: Common Table Expressions vs. Subqueries](#11-query-rewriting-common-table-expressions-vs-subqueries)
13. [A Unified Diagnostic Workflow](#12-a-unified-diagnostic-workflow)
14. [Common Pitfalls](#13-common-pitfalls)
15. [Quick Reference Table](#quick-reference-table)
16. [Conclusion](#conclusion)

---

## Introduction

This series already covers the individual pillars query optimization rests on — indexing (this series' SQL Indexes guide), reading what the optimizer actually decided (this series' Execution Plans guide), join mechanics (this series' Joins guide), and the correctness machinery surrounding concurrent access (this series' Transactions, Deadlocks, and Isolation Levels guides). This guide is the capstone that ties them together into a practical discipline: the specific, recurring ways a query's own *shape* — not missing indexes, not bad data, just how the SQL itself is written — prevents the optimizer from using an index that's sitting right there, ready to help. The central concept is **SARGability**, and once you can recognize it precisely, an enormous fraction of "this query is slow despite having the right index" problems resolve themselves immediately.

```plaintext
CREATE INDEX IX_Orders_OrderDate ON Orders(OrderDate);

-- ❌ NOT SARGable — the index CANNOT be used efficiently, despite existing:
SELECT * FROM Orders WHERE YEAR(OrderDate) = 2026;

-- ✅ SARGable — the SAME logical condition, rewritten so the index CAN be used:
SELECT * FROM Orders WHERE OrderDate >= '2026-01-01' AND OrderDate < '2027-01-01';
```

---

## 1. SARGability: The Single Most Important Concept in This Guide

### What "SARGable" actually means: Search ARGument-able

```plaintext
A predicate is SARGable if the query optimizer can use it DIRECTLY
  against an index's B-tree (per this series' SQL Indexes guide's
  Sections 2-3) to perform a SEEK — navigating straight to matching
  values — rather than needing to evaluate the condition against
  EVERY ROW individually, which forces a SCAN (per this series'
  Execution Plans guide's Section 4) regardless of whether a relevant
  index technically exists.
```

This is the foundational concept this entire guide is organized around, and it's worth understanding precisely why it happens: an index's B-tree is sorted by the *raw column value* — `OrderDate` itself, not `YEAR(OrderDate)`. If your `WHERE` clause asks the optimizer to evaluate a *transformation* of the column (a function call, an implicit type conversion) rather than the column's own stored value directly, the index's sort order becomes useless for that lookup — the database would need to compute that transformation for every single row before it could know whether it matches, which is exactly what a B-tree seek is supposed to let you avoid.

### Why a missing index and a non-SARGable predicate PRODUCE THE SAME SYMPTOM, but need entirely different fixes

```plaintext
Both show up as a Table Scan / Index Scan in the execution plan (per
  this series' Execution Plans guide's Section 4) — the SYMPTOM is
  identical. But "no index exists" is fixed by CREATING one (this
  series' SQL Indexes guide's Section 12); "the predicate isn't
  SARGable" is fixed by REWRITING THE QUERY — adding an index on
  YEAR(OrderDate) as a computed column could theoretically help, but the
  far simpler, more broadly useful fix is almost always to rewrite the
  PREDICATE itself, which is this guide's central, recurring theme.
```

This is worth stating explicitly as the reason this guide exists as its own, distinct topic from indexing — a developer who sees a scan in the plan and concludes "I need an index" without first checking whether the predicate is even SARGable can spend real effort adding an index that never gets used, because the *query*, not the *schema*, was the actual obstacle.

---

## 2. Function-Wrapped Columns: The Most Common SARGability Killer

### Wrapping the INDEXED column in a function makes the index unusable for that predicate

```sql
-- ❌ The function wraps the COLUMN — the optimizer must compute
--    YEAR(OrderDate) for EVERY row before it can compare to 2026
SELECT * FROM Orders WHERE YEAR(OrderDate) = 2026;

-- ❌ Same problem — UPPER() wraps the column
SELECT * FROM Customers WHERE UPPER(Email) = 'ALICE@EXAMPLE.COM';

-- ❌ Same problem — arithmetic on the column
SELECT * FROM Products WHERE Price * 1.1 > 100;
```

Every one of these examples shares the identical structural flaw: the function or expression is applied to the *column*, which means the optimizer cannot simply look up a literal value in the index's sorted B-tree — it has to transform every row's value first, which requires visiting every row, which is precisely a scan.

### The fix: apply the transformation to the CONSTANT/parameter instead, leaving the column bare

```sql
-- ✅ The column itself is untouched — this IS a direct, seekable range predicate
SELECT * FROM Orders WHERE OrderDate >= '2026-01-01' AND OrderDate < '2027-01-01';

-- ✅ Case-insensitive comparison WITHOUT wrapping the column — relies on a
--    case-insensitive COLLATION (common default in SQL Server) rather than UPPER()
SELECT * FROM Customers WHERE Email = 'alice@example.com';

-- ✅ Move the arithmetic to the OTHER side of the comparison
SELECT * FROM Products WHERE Price > 100 / 1.1;
```

This is the single, recurring pattern underneath nearly every SARGability fix in this guide: whatever transformation your query logically needs, apply it to the *literal value or parameter* you're comparing against, never to the column being filtered — the column must appear bare, in its raw, indexed form, for the optimizer to have any chance of seeking against it.

### Why this is so easy to write by accident, especially with dates

```plaintext
Extracting a YEAR, MONTH, or date-truncated value from a datetime
  column is a genuinely common, natural-feeling thing to write — "give
  me everything from this year" reads more directly as YEAR(OrderDate) =
  2026 than as an explicit date RANGE — which is exactly why this
  specific pattern (date-part extraction wrapping the column) is one of
  the most frequently seen SARGability violations in real, production codebases.
```

---

## 3. Implicit Conversions: The Silent SARGability Killer

### A type mismatch between a column and a literal/parameter can wrap the COLUMN in a conversion, invisibly

```sql
-- Products.Sku is NVARCHAR(50) — but the comparison value is NOT prefixed with N'...'
SELECT * FROM Products WHERE Sku = '12345';  -- a VARCHAR literal, compared against an NVARCHAR column
```

This is genuinely the most dangerous variant of Section 2's problem, specifically because nothing in the query's visible text looks like a function call — there's no `YEAR(...)`, no `UPPER(...)` anywhere to spot by eye. But per SQL Server's data type precedence rules, comparing an `NVARCHAR` column against a plain `VARCHAR` literal forces an **implicit conversion** — and because `NVARCHAR` has higher precedence than `VARCHAR`, SQL Server converts the *literal* up to `NVARCHAR`... except when the mismatch runs the other direction (comparing an `INT` column against a string parameter, or specific collation mismatches), the conversion can land on the *column* side instead, silently wrapping every row's value in a `CONVERT()` the execution plan will show but the query's own source code never hints at.

### Why this is particularly common with ADO.NET and ORM-generated parameters

```csharp
// ❌ Using AddWithValue, .NET infers the parameter TYPE from the C# value —
//    a plain C# string often becomes an NVARCHAR parameter by default, which
//    is FINE if the column is NVARCHAR too, but a MISMATCH if the column is VARCHAR
command.Parameters.AddWithValue("@sku", "12345");

// ✅ Explicit parameter typing, matching the ACTUAL column type precisely
command.Parameters.Add("@sku", SqlDbType.VarChar, 50).Value = "12345";
```

This connects directly to how parameterized queries (Section 10) actually reach the database — `AddWithValue`'s type inference is a genuinely well-known, documented source of exactly this implicit-conversion problem, and it's precisely why explicit parameter typing is the standard, recommended practice for any performance-sensitive query, not just a style preference.

### Diagnosing an implicit conversion: it's visible in the execution plan, once you know to look

```plaintext
Per this series' Execution Plans guide's Section 2: an implicit
  conversion shows up as a CONVERT_IMPLICIT expression wrapped around
  the column reference in the plan's XML/properties — and SQL Server's
  graphical plan will often show a WARNING ICON directly on the affected
  operator specifically flagging this, precisely because it's common and
  consequential enough to warrant its own dedicated warning.
```

---

## 4. Leading Wildcards and LIKE

### Why `LIKE '%term'` can't seek, but `LIKE 'term%'` can

```sql
-- ✅ SARGable — a TRAILING wildcard still has a known, fixed STARTING point
--    the B-tree can seek to directly
SELECT * FROM Products WHERE Name LIKE 'Lap%';

-- ❌ NOT SARGable — a LEADING wildcard means there's NO fixed starting
--    point; the match could begin ANYWHERE within the string, so the
--    index's sort order (by the string's FIRST character) offers no help at all
SELECT * FROM Products WHERE Name LIKE '%top';
```

This follows directly from Section 2's B-tree logic — an index on `Name` is sorted by the string's value starting from its first character, exactly the way a phone book is sorted by last name; `'Lap%'` can seek straight to entries beginning with "Lap," exactly like finding every "Smith," but `'%top'` requires checking every single entry's *ending*, which the sort order gives no shortcut for at all, precisely analogous to finding every phone book entry ending in a specific suffix.

### When a leading wildcard is genuinely unavoidable: full-text search, not LIKE

```plaintext
If "find anything CONTAINING this substring, anywhere" is a genuine,
  necessary requirement (not just a convenient way to write a prefix
  search), SQL Server's FULL-TEXT SEARCH feature (CONTAINS/FREETEXT),
  built on its OWN specialized inverted-index structure, is the correct
  tool — forcing a leading-wildcard LIKE to perform well via ordinary
  B-tree indexing is not something query rewriting alone can fix, since
  the underlying problem is a genuine mismatch between the QUERY'S need
  and the B-TREE's fundamental structure.
```

---

## 5. SELECT *: Why It Costs More Than It Looks Like

### The direct cost: fetching columns nobody actually needs

```sql
-- ❌ Fetches EVERY column, including ones the calling code never touches
SELECT * FROM Orders WHERE CustomerId = 42;

-- ✅ Fetches ONLY what's genuinely needed
SELECT Id, OrderDate, Total FROM Orders WHERE CustomerId = 42;
```

This is the obvious cost — more bytes transferred, more memory consumed turning rows into objects — but it's genuinely secondary to the next one.

### The real, structural cost: SELECT * defeats covering indexes entirely

```plaintext
Per this series' SQL Indexes guide's Section 7: a COVERING index
  (INCLUDE-ing specific additional columns) lets a query be satisfied
  ENTIRELY from the index, with NO key lookup back to the full row. A
  query asking for SELECT * can essentially NEVER be covered this way —
  it demands EVERY column the table has, which means a non-clustered
  index could only ever cover it by including EVERY SINGLE COLUMN,
  defeating the entire purpose and cost-benefit of a narrower, targeted covering index.
```

This is worth treating as the real, structural reason `SELECT *` is a genuine performance anti-pattern, not just an imprecision — it doesn't just transfer unneeded bytes; it actively forecloses one of the most effective optimization techniques this series' SQL Indexes guide covers, by making it essentially impossible for any index narrower than the whole table to satisfy the query without a key lookup.

---

## 6. Filtering Early and Precisely

### Pushing a filter as close to the data source as possible, rather than filtering a larger intermediate result

```sql
-- ❌ Joins the FULL Orders and OrderItems tables first, THEN filters —
--    if the optimizer can't reorder this (it often CAN, but not always,
--    especially across views or certain query shapes), more rows than
--    necessary get joined before being discarded
SELECT o.Id, oi.ProductName
FROM Orders o
JOIN OrderItems oi ON oi.OrderId = o.Id
WHERE o.OrderDate >= '2026-01-01' AND o.Status = 'Shipped';

-- The MODERN SQL Server optimizer will typically push this filter down
-- automatically — but when working through a VIEW, a CTE with complex
-- logic, or a query hint that constrains join order, this reordering
-- is not always guaranteed, which is worth KNOWING even though the
-- optimizer handles the common case correctly on its own.
```

Worth knowing that this is less of an active rewriting technique for simple queries (the optimizer is genuinely good at predicate pushdown for straightforward cases) and more of a *diagnostic awareness* — when a query built on top of a view or a complex CTE runs slower than its logical complexity would suggest, checking the execution plan for whether filters were actually pushed down to the earliest possible point, per this series' Execution Plans guide's Section 2 reading discipline, is a genuinely useful thing to verify rather than assume.

### The most selective filter first, when the optimizer needs a hint about join/filter order

```sql
-- Per this series' SQL Indexes guide's Section 8: a HIGHLY selective
-- condition (matching few rows) is generally most valuable to apply
-- as EARLY as possible, since it shrinks the working set the rest of
-- the query has to process
WHERE o.OrderId = @specificOrderId   -- extremely selective, matches ~1 row
  AND o.Status = 'Pending'            -- much LESS selective, matches many rows
```

The optimizer's own statistics-driven cost model (per this series' Execution Plans guide's Section 7) generally handles ordering this correctly on its own in the common case — worth knowing as background understanding of *why* a given plan looks the way it does, more than as a rewriting rule to apply by hand in ordinary circumstances.

---

## 7. Set-Based Thinking vs. Row-by-Row Processing

### The anti-pattern: a cursor, looping over rows and issuing one statement per row

```sql
-- ❌ RBAR ("Row By Agonizing Row") — processes ONE row at a time, with
--    the overhead of a SEPARATE statement execution for EACH
DECLARE order_cursor CURSOR FOR SELECT Id FROM Orders WHERE Status = 'Pending';
OPEN order_cursor;
FETCH NEXT FROM order_cursor INTO @orderId;
WHILE @@FETCH_STATUS = 0
BEGIN
    UPDATE Orders SET Status = 'Processing' WHERE Id = @orderId;
    FETCH NEXT FROM order_cursor INTO @orderId;
END
CLOSE order_cursor;
```

This is the SQL-specific instance of a much more general principle: relational databases are built, from the ground up, to process entire *sets* of rows in a single operation — a cursor defeats this entirely, paying the overhead of statement execution and context-switching once *per row* rather than once for the whole batch.

### The set-based rewrite: one statement, operating on the whole matching set at once

```sql
-- ✅ ONE statement, updating EVERY matching row in a SINGLE operation
UPDATE Orders SET Status = 'Processing' WHERE Status = 'Pending';
```

This is almost always both simpler to write *and* dramatically faster — the database engine's own internal machinery (the same B-tree seeking, locking, and logging this series' SQL Indexes and Transactions guides cover) is designed and optimized around processing a coherent set of rows together, and a cursor forces it to abandon nearly every one of those efficiencies in favor of row-at-a-time, essentially procedural execution.

### Why cursors are sometimes genuinely unavoidable, and how to recognize those cases

```plaintext
A GENUINELY row-dependent operation — where processing row N requires
  the RESULT of having already processed row N-1 (a running calculation
  with complex, non-aggregatable logic, certain recursive hierarchy
  walks a window function can't express) — may legitimately need
  row-by-row processing. The discipline worth applying is confirming
  this dependency is REAL before reaching for a cursor, since the
  overwhelming majority of "I need to loop through these rows" tasks
  turn out to have a genuine, often much simpler set-based equivalent once examined closely.
```

---

## 8. OR Conditions and UNION ALL Rewrites

### Why an OR across different columns can prevent efficient index use

```sql
-- Potentially FORCES a scan, even with indexes on BOTH CustomerId and OrderNumber —
-- the optimizer must find rows matching EITHER condition, and a single index
-- seek can only efficiently serve ONE of them at a time
SELECT * FROM Orders WHERE CustomerId = 42 OR OrderNumber = 'ORD-9981';
```

### The rewrite: UNION ALL, letting EACH branch use its OWN index independently

```sql
SELECT * FROM Orders WHERE CustomerId = 42
UNION ALL
SELECT * FROM Orders WHERE OrderNumber = 'ORD-9981';
-- (UNION ALL, not UNION, if duplicates across the two conditions are
--  known to be impossible or genuinely acceptable — UNION's deduplication
--  step has its own real cost worth avoiding when it isn't needed)
```

Each branch of the `UNION ALL` is its own, independent query, free to use whichever index best serves *its own* single condition — this is a genuinely common, effective rewrite specifically for the "OR across differently-indexed columns" pattern, trading one harder-to-optimize query for two simple, individually SARGable ones combined afterward.

---

## 9. Pagination: OFFSET/FETCH and Keyset Pagination

### OFFSET/FETCH: simple, but genuinely expensive at large offsets

```sql
SELECT * FROM Orders ORDER BY OrderDate DESC
OFFSET 10000 ROWS FETCH NEXT 20 ROWS ONLY;  -- page 501, at 20 rows per page
```

This is worth understanding precisely: `OFFSET` doesn't magically skip to row 10,000 — the engine must still count through (and in many plans, genuinely process) the first 10,000 matching rows in sorted order before it can discard them and return the next 20, which means this approach's cost grows with the offset itself, making deep pagination (page 500+) measurably, increasingly expensive.

### Keyset pagination: using the LAST row's own values as the next page's starting filter

```sql
-- Page 1:
SELECT TOP 20 * FROM Orders ORDER BY OrderDate DESC, Id DESC;
-- note the LAST row's OrderDate and Id from page 1's results, then:
-- Page 2:
SELECT TOP 20 * FROM Orders
WHERE (OrderDate, Id) < (@lastOrderDate, @lastId)   -- a SARGable, SEEKABLE range condition
ORDER BY OrderDate DESC, Id DESC;
```

This is a genuinely different, considerably more scalable approach — rather than asking "skip N rows," it asks "give me the next 20 rows after this specific position," which is precisely the kind of range predicate an index can seek to directly (Section 1), with a cost that stays roughly constant regardless of how deep into the result set you're paging, unlike `OFFSET`'s linearly growing cost.

---

## 10. Parameterization and Plan Reuse

### Why literal values embedded directly in SQL text prevent plan caching

```csharp
// ❌ A DIFFERENT query TEXT for every different customer ID — SQL Server
//    must COMPILE a brand-new plan EVERY time, since it has no way to
//    know these are "the same query" with different inputs
var sql = $"SELECT * FROM Orders WHERE CustomerId = {customerId}";  // ALSO a SQL INJECTION risk

// ✅ The SAME query text, EVERY time — SQL Server recognizes it and
//    reuses the CACHED plan (per this series' Execution Plans guide's Section 10)
var sql = "SELECT * FROM Orders WHERE CustomerId = @CustomerId";
command.Parameters.Add("@CustomerId", SqlDbType.Int).Value = customerId;
```

This connects directly to this series' Execution Plans guide's Section 10 plan-caching discussion — the cache keys on the exact query *text*; a literal-embedded query produces different text (and a different cache entry, paying the full compilation cost) for every distinct input value, while a parameterized query's text never changes, letting the exact same compiled plan be reused across every execution. Beyond the pure performance concern, string-concatenated SQL built from user-controllable input is also, independently, a serious SQL injection vulnerability — parameterization is both a performance *and* a security fix simultaneously.

### The direct link to parameter sniffing — this series' Execution Plans guide's Section 9

```plaintext
Parameterization is what makes plan REUSE possible — and plan reuse is
  PRECISELY what creates the parameter-sniffing risk that guide's
  Section 9 covers in depth: a cached plan, compiled for one parameter
  value, reused for another with a genuinely different data distribution.
  This isn't a contradiction — parameterization is still overwhelmingly
  the right default (its performance and security benefits are real and
  substantial); it's simply worth knowing it trades the compile-every-time
  cost for a DIFFERENT, specific, and separately-manageable risk.
```

---

## 11. Query Rewriting: Common Table Expressions vs. Subqueries

### Why a CTE and an equivalent correlated subquery can produce genuinely different plans

```sql
-- Using a CTE:
WITH RecentOrders AS (
    SELECT CustomerId, COUNT(*) AS OrderCount
    FROM Orders WHERE OrderDate >= '2026-01-01'
    GROUP BY CustomerId
)
SELECT c.Name, r.OrderCount FROM Customers c JOIN RecentOrders r ON r.CustomerId = c.Id;

-- An equivalent CORRELATED subquery:
SELECT c.Name,
    (SELECT COUNT(*) FROM Orders o WHERE o.CustomerId = c.Id AND o.OrderDate >= '2026-01-01') AS OrderCount
FROM Customers c;
```

These two queries are logically equivalent, but the second one's subquery is *correlated* — it references `c.Id` from the outer query, meaning the optimizer may need to re-evaluate it once per outer row (conceptually resembling Section 6's Nested Loops pattern from this series' Execution Plans guide), while the CTE version computes `RecentOrders` as its own, independent, set-based aggregation before joining — worth checking the actual execution plan (never assuming) when these two shapes produce meaningfully different performance for a specific query, since the optimizer's actual ability to rewrite one into the other varies by query complexity.

### CTEs are NOT automatically materialized or cached — a genuinely common misconception

```plaintext
Unlike a TEMP TABLE (which genuinely IS materialized, once, as its own
  physical structure), a CTE is purely a NAMED, reusable piece of query
  TEXT — if the SAME CTE is referenced MULTIPLE times within one query,
  the optimizer may (depending on the specific query and version)
  re-evaluate its underlying logic MULTIPLE times, once per reference,
  rather than computing it once and reusing the result.
```

This is worth knowing precisely because "CTE" and "materialized/cached result" are often assumed to be synonymous, and they genuinely aren't — for a CTE referenced multiple times whose underlying computation is expensive, a temp table (an explicit, one-time materialization) can be a meaningfully better-performing alternative, verified, as always, via the actual execution plan.

---

## 12. A Unified Diagnostic Workflow

### Drawing together this series' SQL Indexes, Execution Plans, Joins, Transactions, Deadlocks, and Isolation Levels guides into one practical process

```plaintext
1. Capture the ACTUAL execution plan (this series' Execution Plans
   guide's Section 1) — never guess.
2. Identify the most expensive operator (that guide's Section 3) and
   check its TYPE — Scan vs. Seek (that guide's Section 4).
3. If it's a Scan where you expected a Seek: check SARGability FIRST
   (Sections 1-4 of THIS guide) — is a function wrapping the column?
   An implicit conversion? A leading wildcard? THIS is very often the
   actual cause, not a missing index.
4. If the predicate genuinely IS SARGable and still scans: check
   whether an appropriate index actually exists (this series' SQL
   Indexes guide), and whether SELECT * (Section 5 of THIS guide) is
   defeating a covering index.
5. Compare ESTIMATED vs. ACTUAL rows (that guide's Section 8) — rule out
   stale statistics as a contributing cause.
6. If JOINS are involved, confirm the join CONDITION itself is correct
   (this series' Joins guide's Section 6's WHERE-vs-ON caution) and that
   join columns are indexed (this series' SQL Indexes guide, this
   guide's Section 12 of the JOINS guide).
7. If the query is PARAMETERIZED and performance varies by input value,
   consider parameter sniffing (that guide's Section 9).
8. If the query runs inside a LARGER transaction, check whether the
   isolation level (this series' Isolation Levels guide) and
   transaction DURATION (this series' Transactions guide's Section 12)
   are appropriate, and whether this query has shown up in a DEADLOCK
   graph (this series' Deadlocks guide's Section 8-9).
9. Confirm any fix EMPIRICALLY with STATISTICS IO/TIME (this series'
   Execution Plans guide's Section 12) before and after.
```

This is worth treating as the genuine, practical synthesis this whole guide — and in a real sense, this entire SQL sub-series — has been building toward: no single technique in isolation reliably diagnoses a real, production slow query; the actual skill is moving through this sequence methodically, letting each step's finding (or lack of one) determine whether to proceed to the next, rather than guessing at a fix and hoping.

---

## 13. Common Pitfalls

| Pitfall | Why it hurts | Better approach |
|---|---|---|
| Wrapping an indexed column in a function in the WHERE clause | Forces the optimizer to evaluate the function against every row, defeating the index's seek capability entirely | Apply transformations to the literal/parameter instead, leaving the column bare and directly comparable (Section 2) |
| Letting implicit type conversion silently wrap a column | Invisible in the query's own source text, but has the identical scan-forcing effect as an explicit function wrap | Match parameter types exactly to column types; avoid `AddWithValue`'s type inference for performance-sensitive queries (Section 3) |
| Using `SELECT *` out of convenience | Defeats covering indexes structurally, beyond just transferring unneeded bytes | Select only the columns genuinely needed, enabling narrower, more effective covering indexes (Section 5) |
| Using a leading wildcard (`LIKE '%term'`) for substring search | No fixed starting point for the index to seek to, forcing a scan regardless of indexing | Use a trailing wildcard where possible; reach for full-text search when genuine substring matching is required (Section 4) |
| Reaching for a cursor out of habit for bulk row processing | Pays per-row statement overhead the engine's set-based design is specifically built to avoid | Default to a single, set-based statement operating on the whole matching set; reserve cursors for genuinely row-dependent logic (Section 7) |
| Using `OFFSET`/`FETCH` for deep pagination | Cost grows with the offset itself, making later pages increasingly expensive | Use keyset pagination for genuinely scalable, constant-cost paging (Section 9) |
| Concatenating literal values directly into SQL text | Prevents plan reuse (forcing recompilation every execution) and is a serious SQL injection risk | Always parameterize; never embed user-controllable (or even internal) values directly into query text (Section 10) |
| Assuming a CTE is automatically materialized/cached | A CTE referenced multiple times can be re-evaluated multiple times, unlike a genuinely materialized temp table | Verify via the execution plan; use a temp table explicitly when a single, reusable materialization is genuinely needed (Section 11) |

---

## Quick Reference Table

| Concept | Rule | Fix |
|---|---|---|
| SARGability | Keep the indexed column bare; transform the literal instead | Rewrite `YEAR(Col) = x` as a date range, etc. (Section 2) |
| Implicit conversion | Match parameter types to column types exactly | Explicit parameter typing, never `AddWithValue` for hot paths (Section 3) |
| Wildcards | Leading wildcards can't seek; trailing ones can | Use full-text search for genuine substring matching (Section 4) |
| `SELECT *` | Prevents covering indexes; wastes transfer | Select only needed columns (Section 5) |
| Row-by-row processing | Pays per-row overhead a set operates without | Default to set-based statements; reserve cursors for genuine dependencies (Section 7) |
| Deep pagination | `OFFSET` cost grows with depth | Keyset pagination for constant-cost paging (Section 9) |
| Plan reuse | Literal-embedded SQL can't be cached and reused | Always parameterize (Section 10) |

---

## Conclusion

Query optimization, practiced well, is a diagnostic discipline more than a collection of individual tricks — and SARGability is the single concept that resolves the largest, most common category of "this query is slow despite having the right index" complaints, precisely because it's invisible from the schema's point of view: the index is there, correctly built, and the query simply never asks for it in a form the optimizer can actually use. Recognizing a function-wrapped column, an implicit conversion, or a leading wildcard on sight is a learnable, mechanical skill, exactly like reading an execution plan, and it's worth developing specifically because it's so often the actual root cause hiding behind what looks, at first glance, like a missing or ineffective index.

This guide's real purpose is Section 12's unified workflow — not because any single technique here is novel on its own, but because knowing the correct *order* to check things in (SARGability before assuming a missing index; statistics staleness before assuming a logic bug; isolation level and transaction duration before assuming the query itself is the whole problem) is what turns this series' SQL Indexes, Execution Plans, Joins, Transactions, Deadlocks, and Isolation Levels guides from five separate bodies of knowledge into one coherent, practical, end-to-end skill for making a slow query fast and keeping it that way.

---

*Found this useful? Feel free to star the repo, open an issue with corrections, or share the index-existed-the-whole-time-but-YEAR()-was-wrapping-the-column discovery that made SARGability click far better than any B-tree diagram ever could.*
