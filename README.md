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