# 数据库

## 自动升级刷库

1. 自动维护管理数据库的版本信息，记录当前服务数据库版本。
2. 支持实现连续多版本自动升级。

### 幂等性

> 应对：手动重跑、测试环境反复执行、脚本复制调试等意外场景。
>
> Oracle 幂等需要 PL/SQL 代码块实现

```sql
-- DDL
-- 建表
CREATE TABLE IF NOT EXISTS t_user {};
-- 新增字段
ALTER TABLE t_user ADD COLUMN IF NOT EXISTS phone VARCHAR(32);
-- 删除字段
ALTER TABLE t_user DROP COLUMN IF EXISTS phone;
-- 新增索引 / 删除索引类似
CREATE INDEX IF NOT EXISTS idx_user_name ON t_user(name);
-- 删表
DROP TABLE IF EXISTS t_user;

-- DML
-- 插入，MYSQL
INSERT IGNORE INTO t_user (id, name) VALUES (1, 'Alice') 
-- 插入，PG
INSERT INTO t_user (id, name) VALUES (1, 'Alice') ON CONFLICT (id) DO NOTHING;
-- 修改/删除（通过WHRER限定）：只会修改匹配数据；重复执行无副作用
UPDATE t_user SET status=1 WHERE id=100;
```



### 自定义实现

核心业务实现逻辑：

- 版本对应的 SQL 代码文件

  ```json
  { 
    "version": "1.0", //按版本号执行
    "sql": [ //按序执行对应地址的sql脚本
      "/v1.0/001-createDataBase.sql",
      "/v1.0/002-initDataBase.sql"
    ]
  }
  ```

- 定义数据库连接配置

- 从数据库中 version 表读取当前的版本号

  - 如果version表不存在，则创建 version 表，设置初始版本 0.0；
  - 否则，获取 version 表的版本号字段对应的值

- 循环判断 sql_version.json 中获取所有的版本号

  - 如果 json版本号 > 表中的版本号，则按序执行对应的sql语句，如3.0版本的时候，不需要执行2.0的sql
  - sql执行全部成功，则返回true，否则以异常的形式抛出，整体执行（或每个SQL文件）作为一个事务；

- 单文件的多 SQL 可通过 `;`或`/` 进行分隔；



### 开源

[Flyway](https://github.com/flyway/flyway)：Java语言，支持 `SQL/Java` 代码（自动扫描）进行版本迁移，开源版不支持回滚；

- Java 迁移代码放入`src/main/java/db/migration/V003__AddCreatedAtColumn.java`下；
- `V${version}__${description}.sql/java`：双下划线分割版本和描述信息，默认校验和。

```java
// 通过 classpath 下的 db/migration（默认） 目录下搜索对应的 java/sql 代码
Flyway flyway = Flyway.configure()
        .baselineOnMigrate(true)      // 发现无历史表自动baseline
        .baselineVersion("2.6.0")     // 设定当前存量库基线版本
    	.baselineDescription("Baseline version 2.6.0, existing production schema")
        .defaultSchema("main")
    	.validateOnMigrate(true)
        .dataSource(DB_URL, null, null)
        .locations("classpath:db/migration")
        .load();
```

[liquibase](https://github.com/amacneil/dbmate)：Java语言，支持 `SQL/Java` 代码进行版本迁移，支持回滚；

- changeset 在 yaml 中显式声明，没有约定的版本解析逻辑；

[dbmate](https://github.com/amacneil/dbmate)：GO语言，只支持 `SQL`代码迁移，支持 `up/down`（升级、回滚），[暂不支持 ORACLE](https://github.com/amacneil/dbmate/discussions/365)；





