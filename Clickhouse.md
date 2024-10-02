### What is clickhouse?
Clickhouse is an open source column oriented OLAP database. (Online Analytics Processing).
It is around 1000s time faster as compared to any rdbms (mysql, postgres) for olap queries

Use Cases:
- Log ingesting
- Real time analytics
- Data warehouse

## Overview of clickhouse
ClickHouse is a true column-oriented DBMS. Data is stored by columns, and during the execution of arrays (vectors or chunks of columns). Whenever possible, operations are dispatched on arrays, rather than on individual values.

In traditional db a row is the smallest unit stored together. But in clickhouse you ingest row, but internally
it stored a new data file for each column. This in turn helps reduces the storage as we can apply diff compression
techniques based on datatype.

## How clickhouse is faster as compared to traditional rdbms
**Example**: `Select avg(price) from products`

In a traditional row-based RDBMS, data is stored row by row. When you execute a query like SELECT avg(price) FROM products, the database engine typically loads entire rows into memory, even though you are only interested in the price column

**ClickHouse** (Columnar Storage)
  ClickHouse, on the other hand, is a columnar database. In a columnar database, data is stored column by column. This means that when you execute a query like SELECT avg(price) FROM products, ClickHouse only needs to load the price column into memory. This approach has several advantages:

**Efficiency**: 
- **Efficiency** Onlythe necessary columns are read from disk, reducing I/O operations.
- **Compression**: Columns of the same data type are stored together, which allows for better compression rates.
- **Batch Processing**: ClickHouse can process data in batches, which can further improve performance.

## Table Functions Clickhouse
Table functions are methods for constructing tables. You can create table from postgres, S3 bucket file and much more.
Supported table functions: https://clickhouse.com/docs/en/sql-reference/table-functions

## Table Engines:

### Merge Tree
- Everytime we insert something in a table its creates a part which contains the column file and the index. (each part is an immutable file) So if I insert 8 rows it will create 8 parts which sits in their individual folders 
- Behind the scenes clickhouse automatically merge these folders so you don't have too many folders.
- Due to this reason it is recommended to always insert in bulk in clickhouse, or use `async insert`.
- Clickhouse can even handle inserting a million rows at a time.
- Max limit of a merge part in clickhouse 150GB

Notes:
- Merge tree is optimized for insertion and read, but not updates or deletes
- We can run update and delete query using `Alter` command which in turn create mutations. You get the response back
but the actual delete or update operation will happen sometime in the future. 

There are other ways to update or data and clickhouse provider diff table engines for that

### Replacing merge Tree
- Remove duplicate entries with same sorting key. If two rows have the same sort key that is last one is persisted
- Deletion of duplicated rows will happen sometime in the future when next merge occurs
- Do get the final result after merging we can use `Final`. During select query clickhouse merges the data

### Collapsing Merge Tree
- Collapses pairs of rows if all the keys in sorting keys are same
- It uses sign field / column -1 or 1 for update and delete.
- If the last inserted row have sign of -1, it means deletion and vice versa for updates

### Version Collapsed Merge Tree
- Both replacing and collapsing merge tree work on the last row inserted based on timing. But there are some use case where
timing can be diff in case of a race condition.
- It uses both sign and version field
- Rows are deleted if they have same sort key and version but diff sign.



**Granule**: A granule is the smallest no of rows that clickhouse reads when searching for rows. Default: 8192 rows

#### Primary Key And Sorting Key
- Primary key is used primarily for indexing
- The ORDER BY clause in ClickHouse is used to define the order in which data is physically stored on disk.
- Primary key in clickhouse is not unique.
- If no primary key is not defined in that case primary Key = sorting key (order by)
- The primary key needs to be a prefix of the sorting key if both are specified.

### What should be the primary key for the table?
If most of the queries work on column A then you should put A early on in the primary key. Also you want to add the values with 
low cardinality values earlier as this will help to skip as many granules as possible. In order words
primary keys ordered by cardinality in ascending order.

#### Primary Index in Clickhouse
- In clickhouse primary idx in sparse not dense. Each entry in primary idx is the first entry of granule
- This behaviour allows clickhouse to store the entire primary idx in memory.

**Note**: That's why it is recommended to use a primary key which few column as clickhouse load the entire primary idx in memory


### Secondary or skipping index
- As the name suggests a skip indx is used to skip as many granules as possible. They don't store the data twice or sort the data differently.
- A skipping idx can only be built on existing sort order of data

Examples of skip-idx:
Set: store uniq value per granule
MinMax: min and max value for a granule



### Read Lifecycle:
- Find the possible granules for each column to grab that might contain the data.
- Decompress the granule
- Process and return the result.

So if a query needs to process 10 granules, it uses all the cores available and processes these granules concurrently. This is yet
another reason for why Clickhouse is so fast.

### Datatypes in clickhouse
- Uint8, Int16
- Float32, Decimal
- String, FixedString(N)
- Date, Date32, Date64
- Enum, LowCardinality, Array, Map, UUID

#### DataFormats: https://clickhouse.com/docs/en/integrations/data-formats

- Nullable types cannot be part of primary key. Always try to avoid nullables values as it creates a hidden column that represents 0,1
- Always use enums for columns with low cardinality values as behind the scenes it is stored as integers
- LowCardinality is similar to Enum. (You don't need to specify value at table creation, dynamically add new values)

### Materialized Views
A materialized views are insert triggers that stores the data of a qiery inside
another destination table.
 
 Use cases: As we know a table in clickhouse can only have 1 primary key, what if we want to query on a primary 
key column? The answer is create a materialized view and change the primary key.

Limitations:
- Only supports insert triggers, updated and deletes are not synced

### Projections
- A projection is like a materialized view for part folder.
- Useful if you want to run query on a non-primary key
- A projection may or may not be used, clickhouse decides that
- We can even add projection after table is created
- Projection is only applied to new rows inserted. In order to populated the projection for all parts you can use
`Materialize` the projection. (Materializing projection also creates a mutation)
- Projection behind the scenes uses mergeTree storage engine.
- Projection increase the time complexity for insertions and merges

### [Joins](https://clickhouse.com/blog/clickhouse-fully-supports-joins-direct-join-part4)
Just like any other rdbms clickhouse supports joins too.
Algorithms:
- **hash**:  creates in memory hash table for right table
- **parallel hash**: similar to hash splits table in multiple hash and loads in memory
- **grace hash**: similar to hash but doesn't need to fit in memory (limits data in memory)
- **partial merge**: variant of sort merge minimizes memory usage
- **full sorting merge**: classing sort merge join
- **direct (default)**: right table is stored as key value in memory using dictionary or join table engine

Sort Merge: both the tables are sorted by join key first in memory if possible other spills to disk
hash and full sorting merge take similar time

Notes:
- In terms of speed direct is fastest with decent memory usage
- In terms of memory hash and parallel hash are with highest memory usage
- In general always put the smaller table on the right



