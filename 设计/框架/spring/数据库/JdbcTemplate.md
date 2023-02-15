## 插入数据

```java
@Autowired
private JdbcTemplate jdbcTemplate;

// v1
jdbcTemplate.update("insert into user(name, create_time) values (?, ?)",
                    "jack", new Date());

// v2
Object[] params = new Object[]{"jack", new Date()};
jdbcTemplate.update("insert into user(name, create_time) values (?, ?)", params);


// 如果希望将一个 List 中的数据批量更新到数据库中，getBatchSize 设置为 List 的大小
// 如果 List 非常大，希望多次批量提交，可以分段读取将大的 List 暂存到小 List 中，再将这个小 List 批量保存到数据库中
jdbcTemplate.batchUpdate("insert into user(name, create_time) values (?, ?)", new BatchPreparedStatementSetter() {
    @Override
    public void setValues(PreparedStatement preparedStatement, int i) throws SQLException {
        preparedStatement.setString(1, userList.get(i).getName());
        preparedStatement.setTimestamp(2, new Timestamp(userList.get(i).getCreateTime().getTime()));
    }

    @Override
    public int getBatchSize() {
        return userList.size();
    }
});
```

```java
@Autowired
private NamedParameterJdbcTemplate jdbcTemplate;

// v1
jdbcTemplate.update("insert into user(name, create_time) values (:name, :createTime)",
                    userMap);

// v2
SqlParameterSource parameterSource = new BeanPropertySqlParameterSource(user);
jdbcTemplate.update("insert into user(name, create_time) values (:name, :createTime)",
                    parameterSource);

// v3
SqlParameterSource parameterSource = new MapSqlParameterSource();
((MapSqlParameterSource) parameterSource).addValue("name", user.getName());
((MapSqlParameterSource) parameterSource).addValue("createTime", user.getCreateTime());
jdbcTemplate.update("insert into user(name, create_time) values (:name, :createTime)",
                    parameterSource);


jdbcTemplate.batchUpdate("insert into user(name, create_time) values (:name, :createTime)",
                         SqlParameterSourceUtils.createBatch(userList));
```

## 查询数据

```java
@Autowired
private JdbcTemplate jdbcTemplate;

jdbcTemplate.queryForObject("select count(*) from user", Long.class);

Object[] params = new Object[]{id};

User user = jdbcTemplate.queryForObject("select * from user where id = ?", params, new RowMapper<User>() {
    @Override
    public User mapRow(ResultSet resultSet, int i) throws SQLException {
        User u = new User();
        u.setId(resultSet.getInt(0));
        u.setName(resultSet.getString(1));
        u.setCreateTime(resultSet.getTimestamp(2));
        return u;
    }
});

List<String> names = jdbcTemplate.queryForList("select name from user", String.class);

// 当处理大结果集时，如果使用 RowMapper<T> 接口，采用的方式是将结果集中的所有数据放到 List 对象中，会占用大量 JVM 内存
List<User> userList = jdbcTemplate.query("select * from user", new RowMapper<User>() {
    @SuppressWarnings("Duplicates")
    @Override
    public User mapRow(ResultSet resultSet, int i) throws SQLException {
        User user = new User();
        user.setId(resultSet.getInt("id"));
        user.setName(resultSet.getString("name"));
        user.setCreateTime(resultSet.getTimestamp("create_time"));
        return user;
    }
});
```

```java
@Autowired
private NamedParameterJdbcTemplate jdbcTemplate;

MapSqlParameterSource parameterSource = new MapSqlParameterSource();
parameterSource.addValue("ids", ids);

List<User> userList = jdbcTemplate.query("select * from user where id in (:ids)",
                                         parameterSource, new RowMapper<User>() {

    @Override
    public User mapRow(ResultSet resultSet, int i) throws SQLException {
        User user = new User();
        user.setId(resultSet.getInt("id"));
        user.setName(resultSet.getString("name"));
        user.setCreateTime(resultSet.getTimestamp("create_time"));
        return user;
    }
});
```