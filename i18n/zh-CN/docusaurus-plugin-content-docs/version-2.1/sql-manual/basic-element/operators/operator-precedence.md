---
{
    "title": "操作符优先级",
    "language": "zh-CN"
}
---

## 描述

操作符优先级决定了在表达式中各个操作符的计算顺序。当表达式中包含多个操作符时，Doris 会根据操作符的优先级从高到低依次进行计算。

## 操作符优先级

优先级从上到下依次递减，最上面具有最高的优先级。

| 优先级 | 操作符                                                       |
| ------ | ------------------------------------------------------------ |
| 1      | !                                                            |
| 2      | +(一元正号), - (一元负号), ~ (一元位反转) ^                  |
| 3      | *, /, %, DIV                                                 |
| 4      | -, +                                                         |
| 5      | &                                                            |
| 6      | \|                                                           |
| 7      | =(比较), <=>, >=, >, <=, <, <>, !=, IS, LIKE, REGEXP, MATCH, IN |
| 8      | NOT                                                          |
| 9      | AND, &&                                                      |
| 10     | XOR                                                          |
| 11     | OR                                                           |
| 12     | \|\|                                                         |