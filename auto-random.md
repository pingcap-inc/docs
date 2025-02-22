---
title: AUTO_RANDOM
summary: Learn the AUTO_RANDOM attribute.
aliases: ['/docs/dev/auto-random/','/docs/dev/reference/sql/attributes/auto-random/']
---

# AUTO_RANDOM <span class="version-mark">New in v3.1.0</span>

> **Note:**
>
> `AUTO_RANDOM` was marked as stable in v4.0.3.

## User scenario

When you write data intensively into TiDB and TiDB has the table with a primary key of the auto-increment integer type, hotspot issue might occur. To solve the hotspot issue, you can use the `AUTO_RANDOM` attribute. Refer to [Highly Concurrent Write Best Practices](/best-practices/high-concurrency-best-practices.md#complex-hotspot-problems) for details.

Take the following created table as an example:

{{< copyable "sql" >}}

```sql
CREATE TABLE t (a bigint PRIMARY KEY AUTO_INCREMENT, b varchar(255))
```

On this `t` table, you execute a large number of `INSERT` statements that do not specify the values of the primary key as below:

{{< copyable "sql" >}}

```sql
INSERT INTO t(b) VALUES ('a'), ('b'), ('c')
```

In the above statement, values of the primary key (column `a`) are not specified, so TiDB uses the continuous auto-increment row values as the row IDs, which might cause write hotspot in a single TiKV node and affect the performance. To avoid such write hotspot, you can specify the `AUTO_RANDOM` attribute rather than the `AUTO_INCREMENT` attribute for the column `a` when you create the table. See the follow examples:

{{< copyable "sql" >}}

```sql
CREATE TABLE t (a bigint PRIMARY KEY AUTO_RANDOM, b varchar(255))
```

or

{{< copyable "sql" >}}

```sql
CREATE TABLE t (a bigint AUTO_RANDOM, b varchar(255), PRIMARY KEY (a))
```

Then execute the `INSERT` statement such as `INSERT INTO t(b) VALUES...`. Now the results will be as follows:

+ Implicitly allocating values: If the `INSERT` statement does not specify the values of the integer primary key column (column `a`) or specify the value as `NULL`, TiDB automatically allocates values to this column. These values are not necessarily auto-increment or continuous but are unique, which avoids the hotspot problem caused by continuous row IDs.
+ Explicitly inserting values: If the `INSERT` statement explicitly specifies the values of the integer primary key column, TiDB saves these values, which works similarly to the `AUTO_INCREMENT` attribute. Note that if you do not set `NO_AUTO_VALUE_ON_ZERO` in the `@@sql_mode` system variable, TiDB will automatically allocate values to this column even if you explicitly specify the value of the integer primary key column as `0`.

> **Note:**
>
> Since v4.0.3, if you want to insert values explicitly, set the value of the `@@allow_auto_random_explicit_insert` system variable to `1` (`0` by default). This explicit insertion is not supported by default and the reason is documented in the [restrictions](#restrictions) section.

TiDB automatically allocates values in the following way:

The highest five digits (ignoring the sign bit) of the row value in binary (namely, shard bits) are determined by the starting time of the current transaction. The remaining digits are allocated values in an auto-increment order.

To use different number of shard bits, append a pair of parentheses to `AUTO_RANDOM` and specify the desired number of shard bits in the parentheses. See the following example:

{{< copyable "sql" >}}

```sql
CREATE TABLE t (a bigint PRIMARY KEY AUTO_RANDOM(3), b varchar(255))
```

In the above `CREATE TABLE` statement, `3` shard bits are specified. The range of the number of shard bits is `[1, 15)`.

After creating the table, use the `SHOW WARNINGS` statement to see the maximum number of implicit allocations supported by the current table:

{{< copyable "sql" >}}

```sql
SHOW WARNINGS
```

```sql
+-------+------+----------------------------------------------------------+
| Level | Code | Message                                                  |
+-------+------+----------------------------------------------------------+
| Note  | 1105 | Available implicit allocation times: 1152921504606846976 |
+-------+------+----------------------------------------------------------+
```

> **Note:**
>
> Since v4.0.3, the type of the `AUTO_RANDOM` column can only be `BIGINT`. This is to ensure the maximum number of implicit allocations.

In addition, to view the shard bit number of the table with the `AUTO_RANDOM` attribute, you can see the value of the `PK_AUTO_RANDOM_BITS=x` mode in the `TIDB_ROW_ID_SHARDING_INFO` column in the `information_schema.tables` system table. `x` is the number of shard bits.

Values allocated to the `AUTO_RANDOM` column affect `last_insert_id()`. You can use `SELECT last_insert_id ()` to get the ID that TiDB last implicitly allocates. For example:

{{< copyable "sql" >}}

```sql
INSERT INTO t (b) VALUES ("b")
SELECT * FROM t;
SELECT last_insert_id()
```

You might see the following result:

```
+------------+---+
| a          | b |
+------------+---+
| 1073741825 | b |
+------------+---+
+------------------+
| last_insert_id() |
+------------------+
| 1073741825       |
+------------------+
```

## Compatibility

TiDB supports parsing the version comment syntax. See the following example:

{{< copyable "sql" >}}

```sql
CREATE TABLE t (a bigint PRIMARY KEY /*T![auto_rand] auto_random */)
```

{{< copyable "sql" >}}

```sql
CREATE TABLE t (a bigint PRIMARY KEY AUTO_RANDOM)
```

The above two statements have the same meaning.

In the result of `SHOW CREATE TABLE`, the `AUTO_RANDOM` attribute is commented out. This comment includes an attribute identifier (for example, `/*T![auto_rand] auto_random */`). Here `auto_rand` represents the `AUTO_RANDOM` attribute. Only the version of TiDB that implements the feature corresponding to this identifier can parse the SQL statement fragment properly.

This attribute supports forward compatibility, namely, downgrade compatibility. TiDB of earlier versions that do not implement this feature ignore the `AUTO_RANDOM` attribute of a table (with the above comment) and can also use the table with the attribute.

## Restrictions

Pay attention to the following restrictions when you use `AUTO_RANDOM`:

- Specify this attribute for the primary key column **ONLY** of `bigint` type. Otherwise, an error occurs. In addition, when the attribute of the primary key is `NONCLUSTERED`, `AUTO_RANDOM` is not supported even on the integer primary key. For more details about the primary key of the `CLUSTERED` type, refer to [clustered index](/clustered-indexes.md).
- You cannot use `ALTER TABLE` to modify the `AUTO_RANDOM` attribute, including adding or removing this attribute.
- You cannot use `ALTER TABLE` to changing from `AUTO_INCREMENT` to `AUTO_RANDOM` if the maximum value is close to the maximum value of the column type.
- You cannot change the column type of the primary key column that is specified with `AUTO_RANDOM` attribute.
- You cannot specify `AUTO_RANDOM` and `AUTO_INCREMENT` for the same column at the same time.
- You cannot specify `AUTO_RANDOM` and `DEFAULT` (the default value of a column) for the same column at the same time.
- When`AUTO_RANDOM` is used on a column, it is difficult to change the column attribute back to `AUTO_INCREMENT` because the auto-generated values might be very large.
- It is **not** recommended that you explicitly specify a value for the column with the `AUTO_RANDOM` attribute when you insert data. Otherwise, the numeral values that can be automatically allocated for this table might be used up in advance.
