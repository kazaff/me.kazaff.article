java新人，问题多多~今天来聊聊JDBC的连接超时问题。

首先，我把mysql的两个相关参数设置为：

	interactive_timeout=10
	wait_timeout=10

单位为秒，也就是说不管是交互还是非交互式的客户端在空闲十秒mysql就会自动断开对应连接。

好，继续，使用`org.springframework.jdbc.datasource.DriverManagerDataSource `来进行数据库查询，是否会出现超时错误呢？

	ApplicationContext context = new ClassPathXmlApplicationContext("spring-config.xml");
    JdbcTemplate jdbcTemplate = (JdbcTemplate)context.getBean("jdbcTemplate");

    String sql = "SELECT * FROM diabloo LIMIT 1";
    User target = jdbcTemplate.queryForObject(sql, new RowMapper<User>() {
        public User mapRow(ResultSet rs, int i) throws SQLException {
            User one = new User();
            one.setName(rs.getString("name"));
            return one;
        }
    });
    System.out.println(target.getName());

    //睡眠足够时间，导致mysql连接超时
    Thread.sleep(11000);

    sql = "SELECT * FROM diabloo LIMIT 1";
    target = jdbcTemplate.queryForObject(sql, new RowMapper<User>() {
        public User mapRow(ResultSet rs, int i) throws SQLException {
            User one = new User();
            one.setName(rs.getString("name"));
            return one;
        }
    });
    System.out.println(target.getName());

我以为会超时的，但却一切正常，后来gg搜索，才发现原因：

> DriverManagerDataSource建立连接是只要有连接就新建一个connection，......

这在我看来，有点吃惊，不过后来就明白了Spring提供这个类的目的，我想更多的理由是用它来充当数据池的**connectFactory角色**。

再看来直接使用原始的JDBC情况下是否会出现超时：

	Class.forName("com.mysql.jdbc.Driver");
    // 建立连接
    Connection con = DriverManager.getConnection(
            "jdbc:mysql://localhost:3306/test", "root", "");
    Statement stmt = con.createStatement();
    ResultSet rs = stmt.executeQuery("SELECT * FROM diabloo LIMIT 1");
    while (rs.next()) {
        String name = rs.getString("name");
        System.out.println(name);
    }

    //睡眠足够时间，导致mysql连接超时
    Thread.sleep(11000);

    stmt = con.createStatement();
    rs = stmt.executeQuery("SELECT * FROM diabloo LIMIT 1");
    while (rs.next()) {
        String name = rs.getString("name");
        System.out.println(name);
    }

这就和猜想一致了，报超时异常了~~

另附一大神的[文章](http://www.importnew.com/2466.html#show-last-Point)，推荐阅读~