---
{
    "title": "ADMIN-SET-REPLICA-STATUS",
    "language": "zh-CN"
}
---

## ADMIN-SET-REPLICA-STATUS

### Name

ADMIN SET REPLICA STATUS

## 描述

该语句用于设置指定副本的状态。

该命令目前仅用于手动将某些副本状态设置为 BAD 或 OK，从而使得系统能够自动修复这些副本

语法：

```sql
ADMIN SET REPLICA STATUS
        PROPERTIES ("key" = "value", ...);
```

 目前支持如下属性：

1. "tablet_id"：必需。指定一个 Tablet Id.
2. "backend_id"：必需。指定 Backend Id.
3.  "status"：必需。指定状态。当前仅支持 "bad" 或 "ok"

如果指定的副本不存在，或状态已经是 bad，则会被忽略。

> 注意：
>
>  设置为 Bad 状态的副本可能立刻被删除，请谨慎操作。

## 举例

 1. 设置 tablet 10003 在 BE 10001 上的副本状态为 bad。

```sql
ADMIN SET REPLICA STATUS PROPERTIES("tablet_id" = "10003", "backend_id" = "10001", "status" = "bad");
```

2. 设置 tablet 10003 在 BE 10001 上的副本状态为 ok。

```sql
ADMIN SET REPLICA STATUS PROPERTIES("tablet_id" = "10003", "backend_id" = "10001", "status" = "ok");
```

### Keywords

    ADMIN, SET, REPLICA, STATUS

### Best Practice

