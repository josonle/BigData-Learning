# 外部数据源API
spark sql针对外部数据源，像json、csv、parquet、jdbc、hdfs、Hive、S3等等，可以很简单地处理，混合数据源间join操作，也可以指定需要的格式存储等等，所以说非常强大

spark sql内置的数据格式有json、parquet、jdbc、csv（2.0之后支持），而外置数据源可以通过引入相关jar包来使用（通过`--jars`引入），原来有一个专门的网站<spark-packages.org>现在已经无法访问了

## 读取Parquet文件
其实就是load方法，`spark.read.format("parquet").load("xxx/xx.parquet")`

不仅仅是parquet，其他格式也是这样读取的，先在format中指明格式，再load进来，区别就是有些可选参数通过option方法设置就行了。当然，也提供了有些快捷方法json()、csv()等，其底层一样还是format再load
> 如果不通过format，只用load加载，其只能加载parquet格式文件（因为默认数据源格式就是parquet）

[更多详细内容可参考我以前的笔记](https://github.com/josonle/Learning-Spark/blob/master/notes/LearningSpark(9)SparkSQL%E6%95%B0%E6%8D%AE%E6%9D%A5%E6%BA%90.md)

看下SQL中怎么加载数据源
```sql
create temporary view parquetTable
# 注意using指明数据源的格式
using org.apache.spark.sql.parquet
options(
  #文件路径
  path "xxx/xx.parquet"
)

spark.sql("select * from parquetTable").show()
```
## 读取JDBC的数据
spark sql这一点特性完全可以取代sqoop，毕竟sqoop也就是在关系型数据库和hdfs、Hive间导进导出，spark sql也可以干，还能直接用来分析，何乐而不为
当然，spark core下也有jdbcRDD，但spark sql这个Data Sources API将远程数据库中的表加载为DataFrame或临时视图，更方便操作

举例spark-shell连接MySQL，[MySQL的驱动jar包下载地址](https://dev.mysql.com/downloads/connector/j/)
```
bin/spark-shell --driver-class-path /xxx/mysql-connector-java-5.1.40.jar --jars /xxx/mysql-connector-java-5.1.40.jar
```

如果是通过spark-submit脚本也可以如下指定驱动
```
# 就是在spark-submit 脚本中加类似下方两个参数
--jars $HOME/tools/mysql-connector-java-5.1.40/mysql-connector-java-5.1.40-bin.jar \
--driver-class-path $HOME/tools/mysql-connector-java-5.1.40-bin.jar \
```

Scala和 SQL的方式连接MySQL

>先说几个可能遇到的问题，要确保Executors都可以访问到JDBC驱动类，也就是mysql那个jar包都可以访问到。其次，连接时最好指明驱动Driver是哪个`option("driver","com.mysql.jdbc.Driver")`

```scala
# read
val jdbcDF = spark.read
  .format("jdbc")
  .option("url", "jdbc:mysql://localhost:3306")
  # 指定数据库下的数据表
  .option("dbtable", "schema.tablename") 
  .option("user", "username")
  .option("password", "password")
  # 最好指明驱动，有可能报错java.sql.SQLException: No suitable driver
  .option("driver","com.mysql.jdbc.Driver")
  .load()

# 也有其他方式read
import java.util.Properties # 需要导入

val connectionProperties = new Properties()
connectionProperties.put("user", "username")
connectionProperties.put("password", "password")
connectionProperties.put("driver","com.mysql.jdbc.Driver")
val jdbcDF2 = spark.read
  .jdbc("jdbc:mysql://localhost:3306", "schema.tablename", connectionProperties)

# save
jdbcDF.write
  .format("jdbc")
  .option("url", "jdbc:mysql://localhost:3306")
  .option("dbtable", "schema.tablename")
  .option("user", "username")
  .option("password", "password")
  .save()

jdbcDF2.write
  .jdbc("jdbc:mysql://localhost:3306", "schema.tablename", connectionProperties)


# SQL的方式连接
CREATE TEMPORARY VIEW jdbcTable
# 像上面说的指定parquet格式一样
USING org.apache.spark.sql.jdbc
OPTIONS (
  url "jdbc:mysql://localhost:3306",
  dbtable "schema.tablename",
  user 'username',
  password 'password',
  driver 'com.mysql.jdbc.Driver'
)
```
option还可以指定很多其他参数，[详情见文档](http://spark.apache.org/docs/latest/sql-data-sources-jdbc.html)
option还可以用单一的options代替，不过内部用Map把参数分装起来
```
options(Map(
  "url" -> "jdbc:mysql://localhost:3306",
  "dbtable" -> "schema.tablename",
  ...省略
))
```
- 写入数据库几点注意项  [参考](https://my.oschina.net/bindyy/blog/680195)

  - 操作该应用程序的用户有对相应数据库操作的权限
  - DataFrame应该转为RDD后，再通过foreachPartition操作把每一个partition插入数据库（不要用foreach，不然每条记录都会连接一次数据库）
