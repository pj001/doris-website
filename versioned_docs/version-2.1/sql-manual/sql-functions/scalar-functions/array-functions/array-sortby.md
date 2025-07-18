---
{
    "title": "ARRAY_SORTBY",
    "language": "en"
}
---

## Description

First, arrange the key column in ascending order, and then return the corresponding column of the src column sorted in this order as the result;
Returns NULL if the input array src is NULL.
If the input array key is NULL, the order in which src is returned remains unchanged.
If the input array key element contains NULL, the output sorted array will place NULL first.

## Syntax

```sql
ARRAY_SORTBY(<src>, <key>)
ARRAY_SORTBY(<lambda>, <arr> [, ...])
```

## Parameters

| Parameter | Description | 
| --- | --- |
| `<lambda>` | A lambda expression where the input parameters must match the number of columns in the given array. The expression can execute valid scalar functions but does not support aggregate functions. |
| `<arr>` | ARRAY array |

## Return Value

Returns a sorted ARRAY type result.

## Example

```sql
select array_sortby(['a','b','c'],[3,2,1]);
```

```text
+----------------------------------------------------+
| array_sortby(ARRAY('a', 'b', 'c'), ARRAY(3, 2, 1)) |
+----------------------------------------------------+
| ['c', 'b', 'a']                                    |
+----------------------------------------------------+
```

```sql
select array_sortby([1,2,3,4,5],[10,5,1,20,80]);
```

```text
+-------------------------------------------------------------+
| array_sortby(ARRAY(1, 2, 3, 4, 5), ARRAY(10, 5, 1, 20, 80)) |
+-------------------------------------------------------------+
| [3, 2, 1, 4, 5]                                             |
+-------------------------------------------------------------+
```

```sql
select *,array_sortby(c_array1,c_array2) from test_array_sortby order by id;
```

```text
+------+-----------------+-------------------------+--------------------------------------+
| id   | c_array1        | c_array2                | array_sortby(`c_array1`, `c_array2`) |
+------+-----------------+-------------------------+--------------------------------------+
|    0 | NULL            | [2]                     | NULL                                 |
|    1 | [1, 2, 3, 4, 5] | [10, 20, -40, 80, -100] | [5, 3, 1, 2, 4]                      |
|    2 | [6, 7, 8]       | [10, 12, 13]            | [6, 7, 8]                            |
|    3 | [1]             | [-100]                  | [1]                                  |
|    4 | NULL            | NULL                    | NULL                                 |
|    5 | [3]             | NULL                    | [3]                                  |
|    6 | [1, 2]          | [2, 1]                  | [2, 1]                               |
|    7 | [NULL]          | [NULL]                  | [NULL]                               |
|    8 | [1, 2, 3]       | [3, 2, 1]               | [3, 2, 1]                            |
+------+-----------------+-------------------------+--------------------------------------+
```

```sql
select *, array_map((x,y)->(y+x),c_array1,c_array2) as arr_sum,array_sortby((x,y)->(y+x),c_array1,c_array2) as arr_sort from array_test2;
```

```text
+------+-----------------+--------------+----------------+-----------------+
| id   | c_array1        | c_array2     | arr_sum        | arr_sort        |
+------+-----------------+--------------+----------------+-----------------+
|    1 | [1, 2, 3]       | [10, 11, 12] | [11, 13, 15]   | [1, 2, 3]       |
|    2 | [4, 3, 5]       | [10, 20, 30] | [14, 23, 35]   | [4, 3, 5]       |
|    3 | [-40, 30, -100] | [30, 10, 20] | [-10, 40, -80] | [-100, -40, 30] |
+------+-----------------+--------------+----------------+-----------------+
```

