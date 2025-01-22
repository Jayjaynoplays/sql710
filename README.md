# Pages, Extents, and Log Files - W3
## Page
- DB stores all data in a **Data Pages** that have **8Kb** each, and this is **immutable**.
- A **page** contains the data that makes up a row.

> If your data row is 4100 bytes long, only a single row will be stored on a page, and the rest
of the page - 3960 bytes - is **wasted** because it can not contain another row

- Type of **Page**:
  - **Data Pages**: Contain actual rows of data.
  - **Text/Image Pages**: For special cases.
  - **Index Pages**: Contain index references of where the data is.
  - **System Pages**: Contains metadata about the organization of the data.
  - _Overflow Page_: We use this when a single row size exceeds 8060 bytes. But still: **a single value cannot exceed the page limit**.

- Each time querying, the DB loads a whole page (8Kb) and then finds the data in that page.
- **A row** of a table **can not be split** in many Data Pages.
- If it contains over 8060 bytes --> moved to the **over-flow page** - _a special page_, **Not split the row in second page**.

### 🧠 Optimize DB method:
> 8192 bytes on a page => approx. 8060 are available to you as a user. <br>
> One part of DB Optimization is optimizing the data type so you can **store as many rows as possible on 1 Page**

Ex: A table may be created with two columns: a **varchar(7000)** and a **varchar(2000)**. 
None of them exceed 8060 bytes, but **combined they** could do so if the entire width of each column is filled.
SQL Server may dynamically move the **varchar(7000)** variable length column to pages in the `ROW_OVERFLOW_DATA` allocation unit.

### ✅ Data Pages Over-flow Consideration:
1. **Dynamic Record Movement**: When records grow beyond the current page size due to updates, they are moved dynamically to a new page. If the record size decreases, they may move back to the original page.
   - Updates that increase a record's size can cause it to exceed the available space on the current page, triggering a relocation to a new page. Conversely, shrinking the record size can allow it to fit back on its original page.

2. **Performance Impact**: Operations like queries, sorts, or joins on **large records with row-overflow data** are slower because they are processed synchronously.
    Large records stored outside the main row (in `ROW_OVERFLOW_DATA` storage) require extra time to fetch and process because they cannot be handled as efficiently as data stored within the row.

3. **Column Length Limits**: Individual columns of types `varchar`, `nvarchar`, `varbinary`, or `sql_variant` must be ≤ 8,000 bytes, but their combined size can exceed the 8,060-byte row limit.
    - While each column has a maximum size limit, their combined length in a row can surpass the table's row size limit because part of the data can be stored off-row.

4. **Other Data Columns**: Columns like `char` and `nchar` must fit within the 8,060-byte row limit, but large object (LOB) data does not count toward this limit.
    - Fixed-length data types (`char`, `nchar`) are restricted to the row size limit, but LOB types (e.g., `text`, `image`) are stored separately and do not affect the main row's size.

5. **Clustered Index Limitations**: A clustered index can't use `varchar` columns with data in the `ROW_OVERFLOW_DATA` allocation unit, and inserting or updating such data may fail.
    - Clustered indexes rely on in-row data for key columns. If a `varchar` column's data is moved to `ROW_OVERFLOW_DATA` storage, it can't serve as a valid key for a clustered index.

6. **Non-clustered Indexes**: Columns with row-overflow data can be included in non-clustered indexes as key or non-key columns.
    - Non-clustered indexes can reference row-overflow data without the same restrictions as clustered indexes, allowing more flexibility in indexing strategies.

## Extents
_TBD_

## Log Files
- Every action to the DB, that statement goes to the transaction log files.
- The log is in binary files so it can not be read by a normal editor, **it's not documented**.
- OS writes the statement to **log files first** before applying to the DB.
- The log file does not use data pages.
- The log file is written sequentially.

#### Personal note:
- Fixed mem space for CHAR(45)
- CHAR is faster than VARCHAR???

---

# Backup and Restore - W3
**Back up and store process must be done at the page level**
- DB Deadlock and prioritize the actions
- Acceptable loss to your company when setting the backup, consider the worst cases.

## Backup
- There are 4 backup types:
  - **Full backup**: Complete copy of the DB at a given time.
  - **Differential backup**: Includes all DB changes since the last full backup.
  - **Transaction Log backup**: The log records each DB transaction, as well as the individual DB modifications made within each transaction.
  - **Copy Only backup**: working as Full backup but does not affect the Differential Change Map (DCM).

> _While performing the backup, you can save it in multiple locations and this will stripe the data making the performance better._

### ✅ Full backup
- The simplest. A full backup is a complete copy of the DB at a given time.
- Can not back up a DB by simply backing up the underlying `.mdf` and `.ldf` (like copying and pasting the data file lollll 🤣).
- **Must use the BACKUP DATABASE command or one of its GUI equivalents.**

#### ⚠️ Caution:
> - When a full backup is restored changes in the full backup file are lost.
> - Even though parts of the transaction log are included in a full backup, this doesn’t constitute a transaction log backup

Query:
```SQL
BACKUP DATABASE [AdventureWorks2012]
TO DISK = 'G:\SQL Backup\AdventureWorks.bak’
WITH INIT
```
Or multi-file backup for performance:
```SQL
BACKUP DATABASE [ADVENTUREWORKS2012]
TO DISK = ‘G:\SQL BACKUP\ADVENTUREWORKS_1.BAK’,
DISK = ‘H:\SQL BACKUP\ADVENTUREWORKS_2.BAK’,
DISK = ‘I:\SQL BACKUP\ADVENTUREWORKS_3.BAK
WITH INIT
```

You can perform backups in SQL Server while the DB is in use and is being modified by users:
<img width="721" alt="Image" src="https://github.com/user-attachments/assets/26a79148-90fa-467e-81bd-ff7f08bb7936" />

- Transaction A **completed** before the backup process completely reading the **Data Pages**, therefore Transaction A
will be in the backup file and will be restored.
- Transaction B **never finished** before the backup process completely reading the **Data Pages** that not committed 
transaction will **not be rolled back** when DB does the restoration process.

### 👉 Restore a full backup
```SQL
RESTORE DATABASE [AdventureWorks2012]
FROM DISK = ‘G:\SQL Backup\AdventureWorks.bak'
WITH REPLACE
```

### ✅ Differential backup:
Includes all DB changes since the last full backup.

Query:
```SQL
BACKUP DATABASE [AdventureWorks2012]
TO DISK = ‘G:\SQL Backup\AdventureWorks-Diff.bak'
WITH DIFFERENTIAL, INIT
```

### 👉 Restore a differential backup
An example of a backup process was designed as weekly full backups and daily differential
backups:

<img width="690" alt="Image" src="https://github.com/user-attachments/assets/792ccdd8-810a-493e-94b4-daa9c7031e6b" />

In the example above, if we needed to restore the DB on Friday morning:
- Restore the full backup first from Sunday.
- Then, restore the different backups on Thursday.
- **The Tuesday version contains changes in the Monday version, the Wednesday version contains changes in the Tuesday, etc...**

Query:
```SQL
-- Restore the full backup first from Sunday.  
RESTORE DATABASE [AdventureWorks2012]  
FROM DISK = 'G:\SQL Backup\AdventureWorks.bak'  
WITH NORECOVERY, REPLACE  
GO  
  
--The, restore the different backup on Thursday.  
RESTORE DATABASE [AdventureWorks2012]  
FROM DISK = ‘G:\SQL Backup\AdventureWorks-Diff.bak'  
GO  
```

#### ✍️ Note:
> - If only using full backups, when a full backup is restored changes since the full backup is lost.
> - Differential backups grow in size and duration the further they are from their corresponding full backup.


### ✅ Transaction Log backup:
- **🧠 The backup for log files 🧠**
- Contains every DB transaction including the modifications made within each transaction.
- If a transaction is canceled before it completes, the transaction log is used for rolling back the transaction’s modifications.
- Transaction logs continue to grow in size until **a transaction log backup is done**.

Query:
```SQL
BACKUP LOG [AdventureWorks2012]
TO DISK = 'G:\SQL Backup\AdventureWorks-trn.bak'
WITH INIT
```
#### ⚠️ Considerations:
- **Frequent transaction log backups** minimize data loss: 
  - If a log backup is taken every 15 minutes, the maximum data loss is 15 minutes, **imagine we back up the log every 1 day**. 
  - **1,440 log backups** would need to be restored if backups are taken every minute, with the last full backup 24 hours ago.
- **Balance** is key between **data loss** and **restore complexity**, influenced by:
    - Database change rate.
    - Maximum allowable data loss.

#### ✍️ Note:
> If only using **full backups** and **differential backups** you can only restore changes made to the
   database since the differential backup will be lost.

#### 👉 Log chain:
Each transaction log backup forms part of what’s called a **log chain**. The head of a log chain is a full
database backup, performed after the database is first created, or when the database’s recovery model
is changed.

<img width="740" alt="Image" src="https://github.com/user-attachments/assets/15062fe0-d036-4aed-8867-232bd69a5da5" />

- To restore a database to a specific point in time, an **unbroken chain of transaction logs** is required, starting from a full backup and continuing through all subsequent log backups.
    - Example: If backup 4 is missing, you can only restore up to the end of backup 3 (e.g., 6 a.m. Tuesday). Any attempt to restore beyond backup 3, such as using log backup 5, will result in an error.

- **Benefits of regular log backups:**
    - They **protect against data loss** by providing recovery points.
    - They **limit the growth of the transaction log file** by removing older log records and freeing up space for new transactions.

- In **full recovery mode**, the transaction log will **grow indefinitely** if no log backups are taken, eventually consuming all available storage. Regular backups prevent this issue.

Query:
```SQL
-- Backup the tail of the transaction log
BACKUP LOG [AdventureWorks2012]
TO DISK = ‘G:\SQL Backup\AdventureWorks-Tail.bak'
WITH INIT, NORECOVERY

-- Restore the full backup
RESTORE DATABASE [AdventureWorks2012]
FROM DISK = 'G:\SQL Backup\AdventureWorks.bak'
WITH NORECOVERY
GO

-- Restore the differential backup
RESTORE DATABASE [AdventureWorks2012]
FROM DISK = 'G:\SQL Backup\AdventureWorks-Diff.bak'
WITH NORECOVERY
GO

-- Restore the transaction logs -------- REPEAT this for all transaction logs -------------------
RESTORE LOG [AdventureWorks2012]
FROM DISK = 'G:\SQL Backup\AdventureWorks-Trn.bak'
WITH NORECOVERY
GO

-- Restore the final tail back up, stopping at 11.05 AM --- Note: The flag WITH RECOVERY puts the database back online -
RESTORE LOG [AdventureWorks2012]
FROM DISK = 'G:\SQL Backup\AdventureWorks-Tail.bak'
WITH RECOVERY, STOPAT = 'June 24, 2008 11:05 AM'
GO
```

### Restore Options:

| **Option**          | **Definition**                                                                                                                |
|---------------------|-------------------------------------------------------------------------------------------------------------------------------|
| `WITH REPLACE`      | Used if you want to **overwrite an existing DB** with the backup. **If a DB already exists, it will be replaced**.            |
| `WITH NORECOVERY`   | Used if you are restoring a sequence of backups. It keeps the DB in a `restoring` state, so no one or no user can use the DB. |
| `WITH RECOVERY`     | Used to complete the restore process, and bring the DB back online, no longer `restoring` state, user can use DB now.             |

### ✅ Copy Only backup:
**Does not affect the Differential Change Map (DCM)**

**Copy Only backup** supported for both full and transaction log backups, is used in situations
in which the backup sequence shouldn’t be affected.

Example:

<img width="740" alt="Image" src="https://github.com/user-attachments/assets/15062fe0-d036-4aed-8867-232bd69a5da5" />

- **Example Recovery Process:**
    - Full backup on Sunday night.
    - Nightly differential backups and six hourly transaction log backups.
    - To recover to 6 pm on Tuesday: 
      1. restore Sunday’s full backup, 
      2. restore Tuesday’s differential, 
      3. restore the 3 transaction log backups leading to 6 p.m.

**What happens if an additional full backup is made?** <br>
If a developer makes a full backup on Monday morning, the **Tuesday differential restore will fail** because:
- Differential backups use a **Differential Changed Map (DCM)** to track changes since the last full backup.
- The Tuesday differential now depends on the Monday full backup, not Sunday’s.
- The restore fails because the restore sequence does not include the Monday full backup.

👍 If the Monday full backup was done as a **COPY_ONLY backup**, the Tuesday differential would still rely on the Sunday full backup. <br>

👍 A **COPY_ONLY transaction log backup** backs up the log without truncation, keeping the log chain intact without requiring the additional backup file:
```SQL
BACKUP LOG [AdventureWorks2012]
TO DISK = 'G:\SQL Backup\AdventureWorks-Trn_copy.bak'
WITH COPY_ONLY
```
