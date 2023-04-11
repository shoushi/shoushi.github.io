---
title:  mysql大小写敏感
tag: mysql
---

# mysql大小写敏感

1. 查看mysql是否大小写敏感

   ```mysql
   show VARIABLES like 'lower%'
   ```

   | Variable_name          | Value |
   | ---------------------- | ----- |
   | lower_case_file_system | ON    |
   | lower_case_table_names | 1     |

   解析

   * **lower_case_file_system** 		OFF表示大小写敏感，ON表示大小写不敏感
   * **lower_case_table_names**      0代表大小写敏感，1表示不敏感

2. 根据数据库表的字段属性决定是否敏感

   collation 值为 utf8_bin时，采用二进制模式进行存储，大小写敏感

   若为utf8_general_ci时，大小写不敏感