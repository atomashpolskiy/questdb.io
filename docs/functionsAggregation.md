---
id: functionsAggregation
title: Aggregation
sidebar_label: Aggregation
---


## sum

`sum(value)` - adds values.

#### Arguments

- `value` is any numeric value.

#### Description

`sum(value)` adds values ignoring missing data (e.g `null` values).

#### Return value

Return value type is the same as the type of the argument.

#### Examples
- Sum all quantities in the transactions table:
```sql
SELECT sum(quantity) FROM transactions;
```
```
| sum   |
|-------|
| 100   |
```

- Sum all quantities in the transactions table, aggregated by item:
```sql
SELECT item, sum(quantity) FROM transactions;
```

```
| item          | count         |
|---------------|---------------|
| apple         | 53            |
| orange        | 47            |
```

#### Overflow
`sum` does not perform overflow check. To avoid overflow, you can cast the argument to wider type.

For example, with column `a` of type INT, you can use:

```sql
SELECT sum(cast(a as long)) from table
```  






## count

`count()` or `count(*)` - counts rows.

#### Arguments
- `count` does not require arguments.

#### Description
`count()` counts rows, irrespective of underlying data. 

#### Return value
Return value type is `long`.

#### Examples

- Count of rows in the transactions table.
```sql
SELECT count() FROM transactions;
```

```
| count     |
|-----------|
| 100       |
```

- Count of rows in the transactions table aggregated by `payment_type` value. 
```sql
SELECT payment_type, count() FROM transactions;
```

```
| cash_or_card  | count         |
|---------------|---------------|
| cash          | 25            |
| card          | 70            |
| null          | 5             |
```
> Note `null` values are aggregated with `count()`.




## avg

`avg(value)` calculates simple average of values

#### Arguments
- `value` is any numeric value.

#### Description
`avg(value)` averages values ignoring missing data (e.g `null` values).

#### Return value

Return value type is `double`.

#### Examples

- Average transaction amount.
```sql
SELECT avg(amount) FROM transactions;
```

```
| avg       |
|-----------|
| 22.4      |
```

- Average transaction amount aggregated by `payment_type`.
```sql
SELECT payment_type, avg(amount) FROM transactions;
```

```
| cash_or_card  | avg           |
|---------------|---------------|
| cash          | 22.1          |
| card          | 27.4          |
| null          | 18.02         |
```

## min

`min(value)` - finds the lowest value.

#### Arguments
- `value` is any numeric value 

#### Description
`min(value)` finds the lowest value ignoring missing data (e.g `null` values).

#### Return value
Return value type is the same as the type of the argument.

#### Examples

- Lowest amount in the transactions table.
```sql
SELECT min(amount) FROM transactions;
```

```
| min       |
|-----------|
| 12.5      |
```

- Lowest amount in the transactions table, aggregated by `payment_type`.
```sql
SELECT payment_type, min(amount) FROM transactions;
```

```
| cash_or_card  | min           |
|---------------|---------------|
| cash          | 12.5          |
| card          | 15.3          |
| null          | 22.2          |
```





## max

`max(value)` - finds the highest value.

#### Arguments
- `value` is any numeric value 

#### Description
`max(value)` finds the highest value ignoring missing data (e.g `null` values).

#### Return value
Return value type is the same as the type of the argument.

#### Examples

- Highest amount in the transactions table.
```sql
SELECT max(amount) FROM transactions;
```

```
| min       |
|-----------|
| 55.3      |
```

- Highest amount in the transactions table, aggregated by `payment_type`.
```sql
SELECT payment_type, max(amount) FROM transactions;
```

```
| cash_or_card  | amount        |
|---------------|---------------|
| cash          | 31.5          |
| card          | 55.3          |
| null          | 29.2          |
```