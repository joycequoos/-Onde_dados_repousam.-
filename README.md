# Where Data Rests

[← Back to SQL Server Tuning](https://github.com/joycequoos/SQL_Server_Developer_Tuning_Codigoscom_maximo_desempenho./blob/main/README.md)

Introduction to the SQL Server data page: the fundamental storage concepts (Data Pages and Extents) and how the instance uses memory (Buffer Pool, Min/Max Server Memory) — essential knowledge before starting to work with performance.

## Table of Contents

- [Data Pages](#data-pages)
- [Extent](#extent)
- [SQL Server Instance Memory](#sql-server-instance-memory)
  - [Buffer Pool (or Buffer Cache)](#buffer-pool-or-buffer-cache)
  - [Configuring Memory in SQL Server](#configuring-memory-in-sql-server)
  - [Min Server Memory](#min-server-memory)
  - [Max Server Memory](#max-server-memory)
- [Scripts and References](#scripts-and-references)
- [Next Steps](#next-steps)

---

## Data Pages

All data sent from applications and systems to a database is written to tables, through `INSERT` and `UPDATE` statements (for maintenance) or `DELETE` (for deletion).

Internally, tables have another definition, called a **data allocation object**. Each data file has predefined areas where data is written — these areas are associated with allocation objects, and it's within them that data is written in record format. These areas are known as **Data Pages**.

| Characteristic | Description |
| --- | --- |
| **Fundamental unit** | The data page is the smallest allocation unit used by SQL Server, and is the fundamental unit of data storage |
| **Fixed size** | Each data page has exactly **8 KB (8192 bytes)**, divided between header, data area, and control slot |
| **Exclusivity** | A data page is exclusive to a single allocation object, but an allocation object can have several data pages |
| **Usable capacity** | Each row can only store up to **8060 bytes** of data within a page |

To check the space used by a table:

```sql
EXECUTE sp_spaceused 'TableName'
```

> 👇 **To learn more:** official documentation for [`sp_spaceused`](https://docs.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-spaceused-transact-sql).

## Extent

An **Extent** is a logical grouping of data pages, whose purpose is to better manage allocated space. An Extent has exactly **8 data pages**, totaling **64 KB**.

There are two types of Extent:

| Type | Description |
| --- | --- |
| **Mixed Extent** | The data pages belong to different allocation objects |
| **Uniform Extent** | The data pages belong exclusively to a single allocation object |

A new table is initially allocated in a Mixed Extent, using a single data page. If the table needs a new page and the Extent still has unused pages, SQL Server keeps allocating there, alongside pages from other allocation objects. Once there are no more free pages in the Mixed Extent, SQL Server starts allocating all new pages in a Uniform Extent.

> 👇 **Learn more:** [`01 - Page and Extent.sql`](<https://github.com/joycequoos/-Onde_dados_repousam.-/blob/main/01 - Página e Extent.sql>)

## SQL Server Instance Memory

A simple question helps explain why memory matters so much: **is it faster to access data in memory or on disk?** The answer already justifies why SQL Server needs enough memory to handle the data load.

- **The more memory, the better.** It's used to load, from disk, the data that needs to be accessed, keeping it in an area of SQL Server known as the **Buffer Pool**.
- Servers with **16 GB, 32 GB, or 64 GB** cover most demands — but there are installations with more than **512 GB** of memory.
- Microsoft's minimum recommendation is just **1 GB**, but the recommended starting point is at least **4 GB**, always validating with a real analysis of the environment for correct sizing.

### Buffer Pool (or Buffer Cache)

A **buffer** is an **8 KB** area in memory where SQL Server stores the data pages read from allocation objects on disk — a task handled by the **Buffer Manager**.

- The data stays in the buffer until the Buffer Manager needs the area to load new pages. The oldest buffers, and those with modified data, are written to disk and released for new pages.
- When SQL Server needs a piece of data and it's already in the buffer, a **logical read** occurs. If the data is not in the buffer, SQL Server performs a **physical read** from disk into the Buffer Pool.
- The memory area of the Buffer Pool is controlled by two instance settings: **Min Server Memory** and **Max Server Memory**.

### Configuring Memory in SQL Server

When you install SQL Server, it automatically configures the use of the server's available memory, through the **Max Server Memory** and **Min Server Memory** options. To check them:

```sql
EXECUTE sp_configure 'show advanced options', 1
GO
RECONFIGURE WITH OVERRIDE

EXECUTE sp_configure 'min server memory (MB)'
EXECUTE sp_configure 'max server memory (MB)'
```

By default, the result usually shows **1024 KB** of minimum memory and **2147483647 KB** (approximately 2 TB) of maximum memory — the theoretical maximum supported value, not a real usage figure.

### Min Server Memory

- It does **not** represent the minimum memory that SQL Server actually uses.
- When the service starts, it initially allocates **128 KB** and waits for insert, update, and delete activity from the application. As queries run, SQL Server loads data from disk into the memory it has already reserved.
- As long as allocation doesn't exceed the value defined in `Min Server Memory`, that memory belongs to SQL Server and is **not returned** to the operating system, even if requested.
- Once that minimum value is exceeded, SQL Server continues allocating more memory normally — but if the OS needs that memory back, SQL Server can release it, down to the configured minimum limit.

### Max Server Memory

- It does **not** represent the maximum memory that SQL Server actually uses on a day-to-day basis.
- SQL Server keeps allocating data from disk to memory until it reaches the value defined in `Max Server Memory`. Once that limit is reached, if it needs to allocate new data, it writes the oldest data to disk, frees up the corresponding memory area, and then allocates the new data.
- If the operating system (or other applications) needs memory and there isn't enough free memory available, the OS can request it from SQL Server. If the memory reserved by SQL Server isn't in use, it writes the pending data to disk and releases that memory to the OS — down to the limit defined in `Min Server Memory`.

> 📺 **Reference:** [video on the topic on YouTube](https://www.youtube.com/watch?v=OijdLj4lw5c).
>
> 👇 **Learn more:** [`02 - Memory Architecture.sql`](<https://github.com/joycequoos/-Onde_dados_repousam.-/blob/main/02 - Arquitetura da Memoria.sql>)

## Scripts and References

| Script | Content |
| --- | --- |
| [`01 - Page and Extent.sql`](<https://github.com/joycequoos/-Onde_dados_repousam.-/blob/main/01 - Página e Extent.sql>) | Practical examples on Data Pages and Extents |
| [`02 - Memory Architecture.sql`](<https://github.com/joycequoos/-Onde_dados_repousam.-/blob/main/02 - Arquitetura da Memoria.sql>) | Practical examples on SQL Server's memory architecture |

## Next Steps

- Test `sp_spaceused` on real tables in the environment and compare the reserved space with the space actually used by the data.
- Explore the `sys.dm_os_buffer_descriptors` DMV to see, in practice, which pages are currently in the Buffer Pool.
- Document a real case of adjusting `Min Server Memory`/`Max Server Memory` in an environment with multiple instances sharing the same server.
- Connect this content to the next module of the Tuning course, on database design and index structure (B-Tree).
