# PostgreSQL

## How does it process multiple index search?
Reference: https://www.postgresql.org/docs/current/indexes-bitmap-scans.html

Assume column A and B are indexed separately and user create a query like WHERE A = 42 and B = 34, PostgreSQL will do index search twice. After each search, create a bitmap to store the result and do the corresponding AND or OR operation to return the combined results.

- bitmap operation
It processes it at page level and row level. And bitmap has a memory limit size.
TIDBitmap (the overall structure)
  └── pagetable: HashMap<PageNumber, PagetableEntry>
        └── PagetableEntry (per matching page)
              └── words: [uint64, uint64, ...]
                    Each bit = one row slot (line pointer)
                    within that page
TIDBitmap has several PagetableEntry obj, one with exact row match (row bits), one with lossy match (only page match but no row matches when it exceeds the memory limit of the bitmap)

## MVCC (Multi-Version Concurrency Control)
Unlike most other database systems which use locks for concurrency control, Postgres maintains data consistency by using a multiversion model. This means that while querying a database each transaction sees a snapshot of data (a database version) as it was some time ago, regardless of the current state of the underlying data. This protects the transaction from viewing inconsistent data that could be caused by (other) concurrent transaction updates on the same data rows, providing transaction isolation for each database session.

The main difference between multiversion and lock models is that in MVCC locks acquired for querying (reading) data don't conflict with locks acquired for writing data and so reading never blocks writing and writing never blocks reading.

Ref: https://www.postgresql.org/docs/7.1/mvcc.html