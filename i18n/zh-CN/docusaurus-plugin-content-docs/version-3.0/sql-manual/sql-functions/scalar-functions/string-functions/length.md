---
{
    "title": "LENGTH",
    "language": "zh-CN"
}
---

## 描述

返回字符串的字节数。

## 语法

```sql
LENGTH ( <str> )
```

## 参数

| 参数      | 说明            |
|---------|---------------|
| `<str>` | 需要计算字节的字符串    |

## 返回值

字符串 `<str>` 的字节数。

## 举例

```sql
SELECT LENGTH("abc"),length("中国")
```

```text
+---------------+------------------+
| length('abc') | length('中国')   |
+---------------+------------------+
|             3 |                6 |
+---------------+------------------+
```