---
title: 关于 Postgresql zone 变成 0 时区的问题排查
date: 2024-01-14 23:23:43
tags: 
    - Postgresql
    - 问题排查 
---

## 背景
- 使用 Postgresql Jdbc Driver 读取大量数据
```java
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <version>42.6.0</version>
</dependency>
```

- 读取的数据中包含 timestamp with time zone 与 time with time zone
- Pg Server 是 +08:00 时区

<!-- more -->

### 问题描述
```java
public class PostgresqlReadZoneDemo {
    private static final String URL = "jdbc:postgresql://127.0.0.1:xxxx/test";
    private static final String USERNAME = "username";
    private static final String PASSWORD = "password";

    public static void main(String[] args) throws Exception {
        Class.forName("org.postgresql.Driver");
        Connection conn = DriverManager.getConnection(URL, USERNAME, PASSWORD);
        initData(conn);
        printData(conn);
    }

    private static void printData(Connection conn) throws Exception {
        String sqlSelect = "SELECT id, time_with_time_zone, timestamp_with_time_zone "
            + "FROM test_schema.test_time_zone WHERE id >= ? ORDER BY id LIMIT ?";
        int batchSize = 5;
        int lastId = -1;

        try (Statement stmt = conn.createStatement()) {
            stmt.executeUpdate("SET TIME ZONE '+08'");
        }

        int count;
        do {
            count = 0;
            try (PreparedStatement ps = conn.prepareStatement(sqlSelect)) {
                ps.setInt(1, lastId + 1);
                ps.setInt(2, batchSize);
                System.out.println(ps + ", id = " + (lastId + 1) + ", limit = " + batchSize);
                ResultSet rs = ps.executeQuery();
                while (rs.next()) {
                    lastId = rs.getInt("id");
                    System.out.println(rs.getInt("id") + ", " + rs.getString("time_with_time_zone")
                        + ", " + rs.getString("timestamp_with_time_zone"));
                    count++;
                }
                rs.close();
            }
        } while (count > 0);
    }

    private static void initData(Connection conn) throws SQLException {
        createTable(conn);
        insertData(conn);
    }

    private static void createTable(Connection connection) throws SQLException {
        String sqlCreateSchema = "CREATE SCHEMA IF NOT EXISTS test_schema";
        try (PreparedStatement pstmtCreateSchema = connection.prepareStatement(sqlCreateSchema)) {
            pstmtCreateSchema.executeUpdate();
        }

        String sqlDropTable = "DROP TABLE IF EXISTS test_schema.test_time_zone";
        try (PreparedStatement pstmtDropTable = connection.prepareStatement(sqlDropTable)) {
            pstmtDropTable.executeUpdate();
        }

        String sqlCreateTable = "CREATE TABLE IF NOT EXISTS test_schema.test_time_zone (" +
            "id SERIAL PRIMARY KEY," +
            "time_with_time_zone TIME WITH TIME ZONE," +
            "timestamp_with_time_zone TIMESTAMP WITH TIME ZONE)";
        try (PreparedStatement pstmtCreateTable = connection.prepareStatement(sqlCreateTable)) {
            pstmtCreateTable.executeUpdate();
        }
    }

    private static void insertData(Connection connection) throws SQLException {
        AtomicInteger idCounter = new AtomicInteger();
        LocalDateTime now = LocalDateTime.now();
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss.SSS");

        String sql = "INSERT INTO test_schema.test_time_zone (id, time_with_time_zone, timestamp_with_time_zone) VALUES (?, ?, ?)";

        try (PreparedStatement pstmt = connection.prepareStatement(sql)) {
            for (int i = 0; i < 50; i++) {
                int id = idCounter.incrementAndGet();
                LocalTime timeWithTimeZone = now.plusHours(i).toLocalTime();
                LocalDateTime timestampWithTimeZone = now.plusDays(i);

                pstmt.setInt(1, id);
                PGobject timewtz = new PGobject();
                timewtz.setType("timetz");
                timewtz.setValue(timeWithTimeZone.format(DateTimeFormatter.ISO_LOCAL_TIME));
                pstmt.setObject(2, timewtz);
                PGobject timestampwtz = new PGobject();
                timestampwtz.setType("timestamptz");
                timestampwtz.setValue(timestampWithTimeZone.format(formatter));
                pstmt.setObject(3, timestampwtz);
                pstmt.addBatch();
            }
            pstmt.executeBatch();
            System.out.println("All Inserted");
        }
    }
}

```

1. 写入 50 条数据，其中包含 int(主键), time with time zone, timestamp with time zone
2. `set time zone '+08'`
3. 分批次(batchSize = 5)读取数据
```java
SELECT id, time_with_time_zone, timestamp_with_time_zone FROM test_schema.test_time_zone WHERE id >= 0 ORDER BY id LIMIT 5, id = 0, limit = 5
1, 20:56:13.176+08, 2024-01-14 20:56:13.176+08
2, 21:56:13.176+08, 2024-01-15 20:56:13.176+08
3, 22:56:13.176+08, 2024-01-16 20:56:13.176+08
4, 23:56:13.176+08, 2024-01-17 20:56:13.176+08
5, 00:56:13.176+08, 2024-01-18 20:56:13.176+08
SELECT id, time_with_time_zone, timestamp_with_time_zone FROM test_schema.test_time_zone WHERE id >= 6 ORDER BY id LIMIT 5, id = 6, limit = 5
6, 01:56:13.176+08, 2024-01-19 20:56:13.176+08
7, 02:56:13.176+08, 2024-01-20 20:56:13.176+08
8, 03:56:13.176+08, 2024-01-21 20:56:13.176+08
9, 04:56:13.176+08, 2024-01-22 20:56:13.176+08
10, 05:56:13.176+08, 2024-01-23 20:56:13.176+08
SELECT id, time_with_time_zone, timestamp_with_time_zone FROM test_schema.test_time_zone WHERE id >= 11 ORDER BY id LIMIT 5, id = 11, limit = 5
11, 06:56:13.176+08, 2024-01-24 20:56:13.176+08
12, 07:56:13.176+08, 2024-01-25 20:56:13.176+08
13, 08:56:13.176+08, 2024-01-26 20:56:13.176+08
14, 09:56:13.176+08, 2024-01-27 20:56:13.176+08
15, 10:56:13.176+08, 2024-01-28 20:56:13.176+08
SELECT id, time_with_time_zone, timestamp_with_time_zone FROM test_schema.test_time_zone WHERE id >= 16 ORDER BY id LIMIT 5, id = 16, limit = 5
16, 11:56:13.176+08, 2024-01-29 20:56:13.176+08
17, 12:56:13.176+08, 2024-01-30 20:56:13.176+08
18, 13:56:13.176+08, 2024-01-31 20:56:13.176+08
19, 14:56:13.176+08, 2024-02-01 20:56:13.176+08
20, 15:56:13.176+08, 2024-02-02 20:56:13.176+08
SELECT id, time_with_time_zone, timestamp_with_time_zone FROM test_schema.test_time_zone WHERE id >= 21 ORDER BY id LIMIT 5, id = 21, limit = 5
21, 16:56:13.176+08, 2024-02-03 20:56:13.176+08
22, 17:56:13.176+08, 2024-02-04 20:56:13.176+08
23, 18:56:13.176+08, 2024-02-05 20:56:13.176+08
24, 19:56:13.176+08, 2024-02-06 20:56:13.176+08
25, 20:56:13.176+08, 2024-02-07 20:56:13.176+08
SELECT id, time_with_time_zone, timestamp_with_time_zone FROM test_schema.test_time_zone WHERE id >= 26 ORDER BY id LIMIT 5, id = 26, limit = 5
26, 13:56:13.176+00, 2024-02-08 12:56:13.176+00
27, 14:56:13.176+00, 2024-02-09 12:56:13.176+00
28, 15:56:13.176+00, 2024-02-10 12:56:13.176+00
29, 16:56:13.176+00, 2024-02-11 12:56:13.176+00
30, 17:56:13.176+00, 2024-02-12 12:56:13.176+00
SELECT id, time_with_time_zone, timestamp_with_time_zone FROM test_schema.test_time_zone WHERE id >= 31 ORDER BY id LIMIT 5, id = 31, limit = 5
31, 18:56:13.176+00, 2024-02-13 12:56:13.176+00
32, 19:56:13.176+00, 2024-02-14 12:56:13.176+00
33, 20:56:13.176+00, 2024-02-15 12:56:13.176+00
34, 21:56:13.176+00, 2024-02-16 12:56:13.176+00
35, 22:56:13.176+00, 2024-02-17 12:56:13.176+00
SELECT id, time_with_time_zone, timestamp_with_time_zone FROM test_schema.test_time_zone WHERE id >= 36 ORDER BY id LIMIT 5, id = 36, limit = 5
36, 23:56:13.176+00, 2024-02-18 12:56:13.176+00
37, 00:56:13.176+00, 2024-02-19 12:56:13.176+00
38, 01:56:13.176+00, 2024-02-20 12:56:13.176+00
39, 02:56:13.176+00, 2024-02-21 12:56:13.176+00
40, 03:56:13.176+00, 2024-02-22 12:56:13.176+00
SELECT id, time_with_time_zone, timestamp_with_time_zone FROM test_schema.test_time_zone WHERE id >= 41 ORDER BY id LIMIT 5, id = 41, limit = 5
41, 04:56:13.176+00, 2024-02-23 12:56:13.176+00
42, 05:56:13.176+00, 2024-02-24 12:56:13.176+00
43, 06:56:13.176+00, 2024-02-25 12:56:13.176+00
44, 07:56:13.176+00, 2024-02-26 12:56:13.176+00
45, 08:56:13.176+00, 2024-02-27 12:56:13.176+00
SELECT id, time_with_time_zone, timestamp_with_time_zone FROM test_schema.test_time_zone WHERE id >= 46 ORDER BY id LIMIT 5, id = 46, limit = 5
46, 09:56:13.176+00, 2024-02-28 12:56:13.176+00
47, 10:56:13.176+00, 2024-02-29 12:56:13.176+00
48, 11:56:13.176+00, 2024-03-01 12:56:13.176+00
49, 12:56:13.176+00, 2024-03-02 12:56:13.176+00
50, 13:56:13.176+00, 2024-03-03 12:56:13.176+00
```
结果的 timezone 信息分为了两批
`id <= 25`数据是正确的 time zone = `+08:00`
`id > 25`数据是也是正确的，但是 time zone = `+00:00`

## 排查问题思路
### 为什么会是 0 时区？
其实这已经是一个查询到中间，已经定位到是 driver 返回返回数据问题时候的场景了。
这个时候剩余的工作就是抓包
```java
sudo tcpdump -i any -n -w postgres_traffic.pcap 'port 5432'
```
抓到的包在 `postgres_traffic.pcap`中扔到 wireshark 中分析，在比较正确与错误数据的时候可以分析出来点问题
![image.png](https://cdn.nlark.com/yuque/0/2024/png/320642/1705241740865-c3c6739e-fd8b-4e58-823b-d7c5f3c64d94.png#averageHue=%23dadbda&clientId=u5896b1bb-f391-4&from=paste&height=785&id=u615f9dea&originHeight=1570&originWidth=3002&originalType=binary&ratio=2&rotation=0&showTitle=false&size=1177457&status=done&style=none&taskId=u5d7cb4a5-61b1-496a-8ba8-9404edbcfc5&title=&width=1501)
可以从上面解析结果得到(16 进制 byte[] 直接 new String() 是 pg driver 中的文本协议解析逻辑)
```java
20
15:56:13.176+08
2024-02-02 20:56:13.176+08
```
而在错误 timezone 中可以看到要求返回的 result format 格式已经变为 Binary 了
![image.png](/images/pg_time_zone/image_1.png)
结果也是 binary
![image.png](/images/pg_time_zone/image_2.png)
如果将这个结果带入 pg driver 的解析逻辑中，会发现 server 传递的实际上是 UTC timestamp，而 0 时区则是我们本地设置的时区
![image.png](/images/pg_time_zone/image_3.png)
这个时候问题的推断就到了为什么我们本地是 0 时区的问题上来了
通过 `PgResultSet#getString`中代码可以追下去
![image.png](/images/pg_time_zone/image_4.png)
时区在 pgDriver 中是在 `QueryExecutorImpl#receiveParameterStatus`中接收到 param 的 `TimeZone`时候设置的，默认是以本地 JVM 时区为准，然后 `set timezone to '+08'`会通过 `TimestampUtils.parseBackendTimeZone` 设置上
![image.png](/images/pg_time_zone/image_5.png)
问题就出在 `set timezone to '+08'`上
在这里面就会发现实际正确的格式应该是 `set timezone to 'GMT-08'`，如果不带 `GMT`会解析出错，导致最终转换时候成 0 时区数据，而 -08 是 PG 独有的参考 [http://www.postgres.cn/docs/14/datetime-posix-timezone-specs.html](http://www.postgres.cn/docs/14/datetime-posix-timezone-specs.html)
![image.png](/images/pg_time_zone/image_6.png)
时区与实际我们认识到的正好相反，东八区是 -08 时区

### 为什么不是所有的数据都出错？
在上述抓包过程中我们可以看到，实际上如果是使用文本协议来传输的数据是没有问题的，而使用二进制传输的话数据会存在问题。
问题就在影响二进制协议或文本协议的参数在 pg 中的描述二进制协议是默认开启的
![image.png](/images/pg_time_zone/image_7.png)
不断的 Debug 之后在协议交换的时候发现了一些端倪在 `preferQueryMode` 这个参数中
![image.png](/images/pg_time_zone/image_8.png)
当 `preferQueryMode=extended`(默认) 的时候，pg driver 会在执行到五次（默认）相同的 prepareStatement 之后尝试缓存 ps，并将 ps 协议换为 binary 协议与 server 端交互传输，这就导致了在执行五次 SQL 之后返回的结果均是 binary 的协议
![image.png](/images/pg_time_zone/image_9.png)

## 最后怎么解决?
1. `set timezone to 'GMT-8'`
2. 或者可以将协议完全更改为文本协议 `binaryTransfer=false`

## 总结
抓包结合 driver 代码是解决多数复杂数据库问题的良方


