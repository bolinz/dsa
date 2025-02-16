# dsa

要排查并成功删除依赖于某个PostgreSQL账号的对象，请按照以下步骤操作：

### 1. **查找该用户拥有的所有对象**
   使用以下查询生成转移所有权的命令，替换`要删除的用户名`和`new_owner`（如`postgres`）：

   **表、视图、序列等：**
   ```sql
   SELECT 'ALTER ' || 
       CASE relkind
           WHEN 'r' THEN 'TABLE'
           WHEN 'v' THEN 'VIEW'
           WHEN 'm' THEN 'MATERIALIZED VIEW'
           WHEN 'S' THEN 'SEQUENCE'
           WHEN 'f' THEN 'FOREIGN TABLE'
       END || ' ' || nspname || '.' || relname || ' OWNER TO new_owner;'
   FROM pg_class
   JOIN pg_namespace ON pg_namespace.oid = relnamespace
   WHERE relowner = (SELECT oid FROM pg_roles WHERE rolname = '要删除的用户名')
     AND relkind IN ('r', 'v', 'm', 'S', 'f');
   ```

   **函数：**
   ```sql
   SELECT 'ALTER FUNCTION ' || nspname || '.' || proname || '(' || oidvectortypes(proargtypes) || ') OWNER TO new_owner;'
   FROM pg_proc
   JOIN pg_namespace ON pg_namespace.oid = pronamespace
   WHERE proowner = (SELECT oid FROM pg_roles WHERE rolname = '要删除的用户名');
   ```

   **数据库：**
   ```sql
   SELECT 'ALTER DATABASE ' || datname || ' OWNER TO new_owner;'
   FROM pg_database
   WHERE datdba = (SELECT oid FROM pg_roles WHERE rolname = '要删除的用户名');
   ```

   **模式：**
   ```sql
   SELECT 'ALTER SCHEMA ' || nspname || ' OWNER TO new_owner;'
   FROM pg_namespace
   WHERE nspowner = (SELECT oid FROM pg_roles WHERE rolname = '要删除的用户名');
   ```

### 2. **执行生成的ALTER命令**
   将上述查询结果中的`new_owner`替换为实际用户（如`postgres`），并运行这些命令转移所有权。

### 3. **检查依赖关系**
   使用`pg_depend`和`pg_shdepend`查找剩余依赖：

   ```sql
   -- 本地依赖
   SELECT classid::regclass, objid, pg_describe_object(classid, objid, 0) AS object
   FROM pg_depend
   WHERE refobjid = (SELECT oid FROM pg_roles WHERE rolname = '要删除的用户名');

   -- 共享依赖
   SELECT classid::regclass, objid, pg_describe_object(classid, objid, 0) AS object
   FROM pg_shdepend
   WHERE refobjid = (SELECT oid FROM pg_roles WHERE rolname = '要删除的用户名');
   ```

   根据结果处理剩余对象（如调整权限或删除）。

### 4. **删除用户**
   确保所有依赖处理完毕后，执行删除命令：

   ```sql
   DROP USER 要删除的用户名;
   ```

### 注意事项：
- **权限问题**：确保新所有者有适当权限。
- **级联删除**：若对象不再需要，可使用`DROP ... OWNED BY 用户名; CASCADE`删除，但需谨慎。
- **备份**：操作前建议备份数据库。

完成上述步骤后，用户及其依赖应被妥善处理，即可成功删除。

在 PostgreSQL 中，**共享依赖的权限**（通过 `pg_shdepend` 表记录）通常涉及以下类型的对象：

1. **角色成员关系**（用户属于某个角色组）
2. **数据库权限**（用户对某个数据库有特殊权限）
3. **表空间权限**（用户拥有表空间）
4. **全局默认权限**（通过 `ALTER DEFAULT PRIVILEGES` 设置的权限）

以下是排查和处理共享依赖权限的具体方法：

---

### 1. **检查共享依赖的具体对象**
使用以下查询找出所有与目标用户相关的共享依赖：
```sql
SELECT 
  classid::regclass AS dependency_type,
  pg_describe_object(classid, objid, 0) AS object_description,
  deptype AS dependency_relation
FROM pg_shdepend 
WHERE refobjid = (SELECT oid FROM pg_roles WHERE rolname = '要删除的用户名');
```

输出示例：
```
dependency_type | object_description              | dependency_relation
----------------|---------------------------------|--------------------
pg_authid       | role member of admin_group      | n
pg_database     | database mydb (CONNECT)         | o
pg_tablespace   | tablespace mytablespace         | o
```

---

### 2. **处理不同类型的共享依赖**

#### **2.1 角色成员关系**
如果用户属于某个角色组，需将其从角色中移除：
```sql
REVOKE 角色名 FROM 要删除的用户名;
```

#### **2.2 数据库权限**
如果用户拥有数据库的权限（如 `CONNECT`、`CREATE`），需撤销权限：
```sql
REVOKE ALL PRIVILEGES ON DATABASE 数据库名 FROM 要删除的用户名;
```

#### **2.3 表空间权限**
如果用户拥有表空间，需转移所有权或删除表空间：
```sql
ALTER TABLESPACE 表空间名 OWNER TO 新所有者;
```

#### **2.4 全局默认权限**
如果用户设置了全局默认权限（`ALTER DEFAULT PRIVILEGES`），需清除这些设置：
```sql
-- 查看默认权限
SELECT pg_get_userbyid(defaclrole) AS user, 
       defaclobjtype AS object_type,
       pg_get_userbyid(defaclacl[1].grantee) AS granted_to,
       defaclacl[1].privilege_type AS privileges
FROM pg_default_acl
WHERE defaclrole = (SELECT oid FROM pg_roles WHERE rolname = '要删除的用户名');

-- 清除默认权限（直接删除相关条目）
-- 注意：无法直接撤销默认权限，需重新设置或删除相关角色
```

---

### 3. **强制清理依赖（谨慎操作）**
如果确定对象不再需要，可使用 `DROP OWNED BY` 级联删除所有依赖：
```sql
-- 删除用户拥有的所有对象（包括权限）
DROP OWNED BY 要删除的用户名 CASCADE;

-- 再删除用户
DROP USER 要删除的用户名;
```

---

### 4. **最终检查**
再次运行共享依赖查询，确保无残留：
```sql
SELECT * FROM pg_shdepend 
WHERE refobjid = (SELECT oid FROM pg_roles WHERE rolname = '要删除的用户名');
```

若结果为空，即可安全删除用户：
```sql
DROP USER 要删除的用户名;
```

---

### 关键注意事项：
1. **生产环境备份**：操作前务必备份数据库。
2. **权限最小化**：避免未来给用户赋予不必要的全局权限。
3. **级联删除风险**：`DROP OWNED BY` 会删除用户的所有对象，确保数据安全。

通过以上步骤，可彻底清理共享依赖的权限，确保用户成功删除。


在 PostgreSQL 的 `pg_shdepend` 表中，`dependency_relation`（实际字段名是 `deptype`）表示依赖关系的类型。它是一个单字符字段，用于描述**依赖对象**与**被依赖对象**之间的关系。以下是各字符的含义和对应的处理方式：

---

### **`dependency_relation`（`deptype`）的可能值**
| 字符 | 类型                 | 含义                                                                 | 常见场景                                                                 |
|------|----------------------|----------------------------------------------------------------------|--------------------------------------------------------------------------|
| `a`  | `AUTO` (自动依赖)    | 依赖关系由系统自动创建，例如外键约束、继承表等。                     | 外键约束、分区表继承关系。                                               |
| `n`  | `NORMAL` (普通依赖)  | 依赖关系由用户显式创建，例如对象直接引用其他对象。                   | 视图依赖于表、函数依赖于类型。                                           |
| `o`  | `OWNER` (所有者依赖) | 对象的所有权依赖于某个角色（用户）。                                 | 表、数据库、模式的所有者。                                               |
| `p`  | `PIN` (固定依赖)     | 系统内部依赖，用户无法直接操作（例如系统表的依赖）。                 | PostgreSQL 内部系统对象（如 `pg_database`）。                            |
| `r`  | `ACL` (权限依赖)     | 对象的访问控制列表（权限）依赖于某个角色。                           | 用户被授予了表的 `SELECT` 权限。                                         |
| `S`  | `SHARED_DEPENDENCY`  | 共享依赖，表示对象在多个数据库之间共享的依赖（如角色或表空间）。     | 角色成员关系、表空间的所有者。                                           |

---

### **如何解读 `dependency_relation`？**
#### 1. **`o`（OWNER 依赖）**
   - **含义**：目标用户是某个对象的拥有者（Owner）。
   - **处理方式**：需要将对象的所有权转移给其他用户。
   - **示例**：
     ```sql
     ALTER TABLE 表名 OWNER TO 新用户;
     ALTER DATABASE 数据库名 OWNER TO 新用户;
     ```

#### 2. `r`（ACL 依赖）
   - **含义**：目标用户在对象的权限列表（`GRANT`/`REVOKE`）中被显式授权。
   - **处理方式**：撤销该用户对对象的权限。
   - **示例**：
     ```sql
     REVOKE ALL ON TABLE 表名 FROM 要删除的用户;
     REVOKE CONNECT ON DATABASE 数据库名 FROM 要删除的用户;
     ```

#### 3. `S`（SHARED_DEPENDENCY）
   - **含义**：依赖关系涉及全局共享对象（如角色、表空间）。
   - **处理方式**：
     - **角色成员关系**：将用户从角色组中移除。
       ```sql
       REVOKE 角色名 FROM 要删除的用户;
       ```
     - **表空间所有权**：转移表空间所有者。
       ```sql
       ALTER TABLESPACE 表空间名 OWNER TO 新用户;
       ```

#### 4. `n`（NORMAL 依赖）
   - **含义**：对象直接依赖于用户，但需要用户手动处理（例如用户创建的视图或函数）。
   - **处理方式**：删除或修改依赖对象。
   - **示例**：
     ```sql
     DROP VIEW 视图名;  -- 删除依赖对象
     ALTER FUNCTION 函数名 OWNER TO 新用户;  -- 转移所有权
     ```

#### 5. `a`（AUTO 依赖）
   - **含义**：依赖由系统自动创建（如外键约束）。
   - **处理方式**：级联删除依赖对象（谨慎操作）。
   - **示例**：
     ```sql
     DROP TABLE 表名 CASCADE;
     ```

---

### **排查步骤**
1. **运行查询**：通过以下语句查看具体依赖关系：
   ```sql
   SELECT 
     classid::regclass AS 对象类型,
     pg_describe_object(classid, objid, 0) AS 对象描述,
     deptype AS dependency_relation
   FROM pg_shdepend
   WHERE refobjid = (SELECT oid FROM pg_roles WHERE rolname = '要删除的用户名');
   ```

2. **根据 `deptype` 处理依赖**：
   - 若为 `o`（所有者依赖）：转移所有权。
   - 若为 `r`（权限依赖）：撤销权限。
   - 若为 `S`（共享依赖）：清理角色或表空间关联。
   - 若为 `n`/`a`：删除或修改依赖对象。

---

### **示例输出解析**
假设查询结果如下：
```
对象类型   | 对象描述                  | dependency_relation
-----------|---------------------------|--------------------
pg_class  | table public.users        | o
pg_shdepend | role member of admin      | S
pg_database | database mydb (CONNECT) | r
```

- **第一行**：用户是表 `public.users` 的所有者（`o`），需转移所有权。
- **第二行**：用户是角色 `admin` 的成员（`S`），需从角色中移除。
- **第三行**：用户拥有数据库 `mydb` 的 `CONNECT` 权限（`r`），需撤销权限。

---

### **注意事项**
- **谨慎使用 `CASCADE`**：级联删除可能意外删除重要数据。
- **备份数据库**：操作前确保有完整备份。
- **权限最小化**：避免未来赋予用户不必要的全局权限。

通过分析 `dependency_relation`，可以精准定位依赖类型并针对性处理，最终成功删除用户。