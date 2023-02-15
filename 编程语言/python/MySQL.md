```shell
pip install mysql-connector
```

```python
# -*- coding: UTF-8 -*-
import mysql.connector

# 打开数据库连接
db = mysql.connector.connect(
       host="localhost",
       user="root",
       passwd="XXX",
       database='wucai',
       auth_plugin='mysql_native_password'
)

# 获取操作游标
cursor = db.cursor()

# 执行SQL语句
cursor.execute("SELECT VERSION()")

# 获取一条数据
data = cursor.fetchone()
print("MySQL 版本: %s " % data)

# 关闭游标&数据库连接
cursor.close()
db.close()
```

```python
# 插入新球员
sql = "INSERT INTO player (team_id, player_name, height) VALUES (%s, %s, %s)"
val = (1003, "约翰-科林斯", 2.08)
cursor.execute(sql, val)
db.commit()
print(cursor.rowcount, "记录插入成功")
```

```python
# 查询身高大于等于 2.08 的球员
sql = 'SELECT player_id, player_name, height FROM player WHERE height>=2.08'
cursor.execute(sql)
data = cursor.fetchall()
for each_player in data:
  print(each_player)
```

```python
# 修改球员约翰-科林斯
sql = 'UPDATE player SET height = %s WHERE player_name = %s'
val = (2.09, "约翰-科林斯")
cursor.execute(sql, val)
db.commit()
print(cursor.rowcount, "记录被修改")
```

```python
sql = 'DELETE FROM player WHERE player_name = %s'
val = ("约翰-科林斯",)
cursor.execute(sql, val)
db.commit()
print(cursor.rowcount, "记录删除成功")
```