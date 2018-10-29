# 实验2：用户管理 - 掌握管理角色、权根、用户的能力，并在用户之间共享对象。

## 实验内容：
Oracle有一个开发者角色resource，可以创建表、过程、触发器等对象，但是不能创建视图。本训练要求：
- 在pdborcl插接式数据中创建一个新的本地角色con_res_view，该角色包含connect和resource角色，同时也包含CREATE VIEW权限，这样任何拥有con_res_view的用户就同时拥有这三种权限。
- 创建角色之后，再创建用户new_user，给用户分配表空间，设置限额为50M，授予con_res_view角色。
- 最后测试：用新用户new_user连接数据库、创建表，插入数据，创建视图，查询表和视图的数据。

## 实验过程：

我的角色名为con_res_view_mx；用户名为new_user_mx

- 第1步：以system登录到pdborcl，创建角色con_res_view和用户new_user，并授权和分配空间：

```sql
$ sqlplus system/123@pdborcl
SQL> CREATE ROLE con_res_view_mx;
Role created.
SQL> GRANT connect,resource,CREATE VIEW TO con_res_view_mx;
Grant succeeded.
SQL> CREATE USER new_user_mx IDENTIFIED BY 123 DEFAULT TABLESPACE users TEMPORARY TABLESPACE temp;
User created.
SQL> ALTER USER new_user_mx QUOTA 50M ON users;
User altered.
SQL> GRANT con_res_view_mx TO new_user_mx;
Grant succeeded.
SQL> exit
```
> 语句“ALTER USER new_user QUOTA 50M ON users;”是指授权new_user用户访问users表空间，空间限额是50M。

第1步的结果图：
![加载失败](https://github.com/mxbox2/oracle/blob/master/test2/%E7%AC%AC%E4%B8%80%E6%AD%A5.png?raw=true)

- 第2步：我建立的用户new_user_mx连接到pdborcl，创建表mytable和视图myview，插入数据，最后将myview的SELECT对象权限授予hr用户。

```sql
$ sqlplus new_user/123@pdborcl
SQL> show user;
USER is "NEW_USER"
SQL> CREATE TABLE mytable (id number,name varchar(50));
Table created.
SQL> INSERT INTO mytable(id,name)VALUES(1,'zhang');
1 row created.
SQL> INSERT INTO mytable(id,name)VALUES (2,'wang');
1 row created.
SQL> CREATE VIEW myview AS SELECT name FROM mytable;
View created.
SQL> SELECT * FROM myview;
NAME
--------------------------------------------------
zhang
wang
SQL> GRANT SELECT ON myview TO hr;
Grant succeeded.
SQL>exit
```

第2步结果图：

![加载失败](https://github.com/mxbox2/oracle/blob/master/test2/%E7%AC%AC%E4%BA%8C%E6%AD%A5.png?raw=true)

- 第3步：用户hr连接到pdborcl，查询new_user授予它的视图myview

```sql
$ sqlplus hr/123@pdborcl
SQL> SELECT * FROM new_user_mx.myview;
NAME
--------------------------------------------------
zhang
wang
SQL> exit
```
第3步结果图：

![加载失败](https://github.com/mxbox2/oracle/blob/master/test2/%E7%AC%AC%E4%B8%89%E6%AD%A5.png?raw=true)
> 测试一下同学用户之间的表的共享，只读共享和读写共享都测试一下。

只读共享：

授权：

![加载失败](https://github.com/mxbox2/oracle/blob/master/test2/%E5%8F%AA%E8%AF%BB%E6%8E%88%E6%9D%83.png?raw=true)

授权结果：

![加载失败](https://github.com/mxbox2/oracle/blob/master/test2/%E5%8F%AA%E8%AF%BB%E6%8E%88%E6%9D%832.png?raw=true)

读写授权：

授权：

![加载失败](./读写授权.png)

授权结果：

![加载失败](./读写授权2.png)
## 数据库和表空间占用分析

> 当全班同学的实验都做完之后，数据库pdborcl中包含了每个同学的角色和用户。
> 所有同学的用户都使用表空间users存储表的数据。
> 表空间中存储了很多相同名称的表mytable和视图myview，但分别属性于不同的用户，不会引起混淆。
> 随着用户往表中插入数据，表空间的磁盘使用量会增加。

## 查看数据库的使用情况

以下样例查看表空间的数据库文件，以及每个文件的磁盘占用情况。

$ sqlplus system/123@pdborcl
```sql
SQL>SELECT tablespace_name,FILE_NAME,BYTES/1024/1024 MB,MAXBYTES/1024/1024 MAX_MB,autoextensible FROM dba_data_files  WHERE  tablespace_name='USERS';

SQL>SELECT a.tablespace_name "表空间名",Total/1024/1024 "大小MB",
 free/1024/1024 "剩余MB",( total - free )/1024/1024 "使用MB",
 Round(( total - free )/ total,4)* 100 "使用率%"
 from (SELECT tablespace_name,Sum(bytes)free
        FROM   dba_free_space group  BY tablespace_name)a,
       (SELECT tablespace_name,Sum(bytes)total FROM dba_data_files
        group  BY tablespace_name)b
 where  a.tablespace_name = b.tablespace_name;
```
- autoextensible是显示表空间中的数据文件是否自动增加。
- MAX_MB是指数据文件的最大容量。

查看数据库使用情况的结果图：

![加载失败](https://github.com/mxbox2/oracle/blob/master/test2/%E6%9F%A5%E7%9C%8B%E6%95%B0%E6%8D%AE%E5%BA%93%E4%BD%BF%E7%94%A8%E6%83%85%E5%86%B5.png?raw=true)
![加载失败](https://github.com/mxbox2/oracle/blob/master/test2/%E6%9F%A5%E7%9C%8B%E6%95%B0%E6%8D%AE%E5%BA%93%E7%9A%84%E4%BD%BF%E7%94%A8%E6%83%85%E5%86%B52.png?raw=true)
