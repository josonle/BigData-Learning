# CSV文件导入Hive的注意事项

- csv文件一般都有header头（列名），Hive中要设置跳过第一行，`"skip.header.line.count"="1"` 
```sql
create table xxx{
  ...
}
...
tblproperties(
  "skip.header.line.count"="1",  --跳过文件首的1行
  --"skip.footer.line.count"="n"  --跳过文件尾的n行
)
```

- 注意csv文件的列的分隔符时`，`、`；`还是`\t`，如`fields terminated by ','`

- 注意是从本地文件上传，hdfs移动，还是查询已有表插入到新表中，数据导入Hive表的语法不同，分别是`load data local inpath ...`，`load data inpath ...`，`insert into/overwrite table xxx ... select ...`
  - 其实本地上传，也是先把数据复制到hdfs的一个临时目录下（一般是hdfs的home目录下），在从临时目录移动到Hive表对应的数据目录
  - 查询表插入新表时，要确保查询的列数和新表的列数匹配
  - 查询表插入新表的方式适用于分区表、分桶表的数据导入
    ```sql
    --静态分区
    insert into table table1 partition(partition_col="xxx") select ... from query_table;
    --动态分区，要注意分区字段和查询的字段类型是否一致
    insert into table table1 partition(col1) select col1,... from query_table;
    ```
  - 查询的方式还支持同时插入多张新表
    ```sql
    from query_table
    insert into table table1 [partition]
      select col1,col2
    insert into table table2
      select *
    ```
  - `insert into/overwrite`的区别，into是追加，overwrite是重写（先删后写，如果涉及分区的话是重写该分区数据）

- 涉及到分区表插入数据时，要注意是否开启动态分区，是采用严格模式还是非严格的，允许创建的最大动态分区数，涉及以下几个配置
> 
> - `hive.exec.dynamic.partition`，开启动态分区，默认true
> - `hive.exec.dynamic.partition.mode`，动态分区模式，默认是严格的（strict，用户必须指定一个静态分区）。非严格模式（nonstrict）允许所有分区字段都是动态的分区。【记得设置为非严格的】
> - `hive.exec.max.dynamic.partitions`，**总共**允许创建的最大动态分区数，默认1000
> - `hive.exec.max.dynamic.ppartitions.pernode`，**每个节点上**允许创建的最大动态分区数，默认100
> - `hive.exec.max.created.files`，一个MR Job中所有mappers、reducers总共允许创建的最大hdfs文件数，默认100000
> - `hive.error.on.empty.partition`，动态分区为空时是否抛出异常，默认false，也无需设置

- 分区表导入数据时还要注意是静态分区、动态分区，还是混合使用
  - 混合的话，静态分区字段必须写在动态分区字段之前，`partition(pCol="xxx",col1)`