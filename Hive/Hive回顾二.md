# 内置函数及运算符

运算符就哪些加减乘除、逻辑、位运算、is null/is not null、like/rlike(正则匹配)作字符串模糊匹配这些

Array通过下标来取数据（a[0]），Map通过key（m[key]），struct通过属性名（s.x类似结构体中用法）

> 最基本的一个命令，查看函数的描述信息即使用范例，`desc function extended [函数名]`

## 内置函数
### 简单函数（map阶段）
涉及关系运算、数学运算、统计、字符串函数、类型转换、条件函数等，主要提几个

- `nvl(col1,replace_with)/nvl(col1,col2)`，null值填充，如果col1这个列有值为null，可以用默认值replace_with或者col2列的值填充
- `coalesce(T t1, T t2,...)`，非空查找，返回参数中第一个非空值，都为空则返回null
  - 可和nvl一样用，`coalesce(col1,0)`，如果col1为null就返回0（等于用0替换）
- `if(boolean condition, T valueTrue, T valueFalseOrNull)`，如果条件真则返回第二个参数，否则返回第三个
- `case when then else end`
- `concat_ws(String sep,String s1,String s2,...)`，用分隔符sep连接字符串，也可以传入一个字符串数组。如果分隔符是null的话，返回值也是null。也可以传入列col参数，但要求列是字符串类型
- URL解析函数`parse_url`
- JSON解析函数`get_json_object`

### 集合函数（map阶段）
像Arrays、Maps、Structs这类复杂数据类型的创建、访问、处理操作等，今天也学了下Spark SQL提供的处理复杂数据类型的API，挺多相似之处的

- `collect_set(col)`，将列的元素去重后返回一个Array类型字段，（不是一个聚合函数，但因为collect仍可用在查询中，突破了group by限制），常用在列转行
  - 查询非group by字段
  - 可通过`collect_set(col)[0]`下标取出数组中的元素
- `collect_list(col)`，同上只是不去重
> 视频里给出了一道行转列的题，将相同星座和血型的人，以这种格式归类：`射手座,A	  大海|凤姐`
> 
> 孙悟空 白羊座 A
> 大海 射手座 A
> 宋宋 白羊座 B
>猪八戒 白羊座  A
>凤姐 射手座 A
> 
> 所以collect_set可以把同星座血型的人合并成一个array返回，再通过字符串连接即可
>```sql
> select concat(constellation,",",blood) as featurem,concat_ws("|",collect_set(name)) as name_list
> from person
>group by constellation,blood;
>```
- `explode(col)`，将Array类型字段展开成一列，常用在行转列
  - 常和`Lateral View `（侧 视图）结合使用，因为其本身是一个UDTF函数，侧视图常和UDTF结合使用
  - 使用UDTF的查询，查询只能包括单个UDTF，不能包含其他字段或多个UDTF。侧视图也可解决这个问题，所以说常和UDTF结合使用
> 视频中要求将电影类别拆分成多行
《疑犯追踪》  悬疑,动作,科幻,剧情
《Lie to me》 悬疑,警匪,动作,心理,剧情
《战狼 2》  战争,动作,灾难
>
>```sql
>select name,category_name
>from
>    movie lateral view explode(category) tmp as category_name;
>```
> UDTF为每个输入行生成零个或多个输出行，而 Lateral View 先将UDTF应用于基表的每一行，然后将结果输出行连接到输入行，以形成具有所提供的表别名的虚拟表，语法如下：
> `lateral view udtf(expression) tableAlias as columnAlias [, colnumAlias2]`
> - from子句中当然可以有多个Lateral View，而且**后续的LATERAL VIEWS可以引用出现在LATERAL VIEW左侧的任何表格中的列**
>```sql
>SELECT * FROM exampleTable
>LATERAL VIEW explode(col1) myTable1 AS myCol1
>LATERAL VIEW explode(myCol1) myTable2 AS myCol2;
>```
>- 外侧视图（outer lateral view）
>如果使用UDTF没有生成行时，这样查询的话可能省略掉一些值，可以通过outer 来解决该问题，将UDTF返回用null值生成行
>用法同lateral view，只不过是在前面加上了outer关键字（outer lateral view）
>> 参考：<https://www.docs4dev.com/docs/zh/apache-hive/3.1.1/reference/LanguageManual_LateralView.html#%E5%A4%96-lateral-view>



### 聚合函数（reduce阶段）
就像max、min、mean、avg这些常用的

### 特殊函数（窗口函数、分析函数、混合函数）
- `over()`子句，括号内可以指定分区字段（distribute by/partition by）、排序字段（sort by/order by），还可以通过`rows between xxx and xxx`指定窗口大小
  - 窗口大小选项：current row（当前行）、n preceding/following（前/后n行）、unbounded precending/following（到最前起点/最后终点）
  > 具体一点的窗口大小规范语法如下：
  > ```
  > (ROWS | RANGE) BETWEEN (UNBOUNDED | [num]) PRECEDING AND ([num] PRECEDING | CURRENT ROW | (UNBOUNDED | [num]) FOLLOWING)
  > (ROWS | RANGE) BETWEEN CURRENT ROW AND (CURRENT ROW | (UNBOUNDED | [num]) FOLLOWING)
  > (ROWS | RANGE) BETWEEN [num] FOLLOWING AND (UNBOUNDED | [num]) FOLLOWING
  > ```
  > 举例
  > ```
  > BETWEEN 4 PRECEDING AND CURRENT ROW //当前行和前4行（共5行）
  > BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING //当前行到分区结束的所有行
  > BETWEEN 4 PRECEDING AND 1 FOLLOWING //当前行+前4行+后1行（共6行）
  > ```
  - 如果是order by后缺少窗口大小，则默认是`rows between unbounded precending and current row`
  - 如果缺少order by和窗口大小，则默认是`rows between unbounded precending and unbounded following`（即整个表）
  - `聚合函数 +over`，只对前面的聚合函数有效，select子句中可以有多个含窗口的函数。SQL标准中来的，允许所有聚合函数+over构成窗口函数
  - `窗口/分析 +over`
  - Hive 2.1.0后，over子句中也支持聚合函数；Hive 2.1.0后聚合函数支持distinct，但over中有order by或窗口限制时不支持；Hive 2.2.0后，在使用over有order by或者窗口大小限制时，聚合函数支持使用distinct
- 窗口函数
  - `lag(col,n,[default]) +over`，向前取第n条数据（默认n为1），可选参数default表示如果返回结果是null则用default值代替
  - `lead(col,n,[default]) +over`，向后取第n条数据（默认n为1），可选参数default表示如果返回结果是null则用default值代替
  - `first_value/last_value`，分区排序后第一条/最后一条数据
    - `last_value`使用时有个坑，就是**要明确指定窗口大小为整个分区窗口大小**`betwwen unbounded precending and unbounded following` 才能取到正确的最后一条数据，否则它会以unbounded precending到current row为窗口
    > `first_value(id) over(partition by id order by id) first`
    > `last_value(id) over(partition by id order by id) last_no`
    > `last_value(id) over(partition by id order by id between unbounded precending and unbounded following) last`
    > | id  | first | last_no | last |
    > | --- | ----- | ------- | ---- |
    > | 1   | 1     | 1       | 3    |
    > | 2   | 1     | 2       | 3    |
    > | 3   | 1     | 3       | 3    |


- 分析函数
  - `row_number`，返回分区内记录的序列，从1开始，排名不存在相等
  - `rank`，返回分区内该数据的排名，排名相同会在名次中留下空位
  - `dense_rank`，同上，但排名相同不会在名次中留下空位
  > 举例，如下class分区内，score的排序序列
  >|class | score    |row_numbr     |rank     |dense_rank     |
  > |:-:| :-: | :-: | :-: | :-: |
  >|A |  1   |1     |   1  |1     |
  >|A |    2 |  2   |     2|  2   |
  >|A |     2|    3 |     2|    2 |
  >|A |     3|    4 |     4|     3|
  >|B|2|1|1|1|
  >|B|3|2|2|2|
  >|B|4|3|3|3|
  - `percent_rank`，返回 (分区内当前行的rank值-1)/(分区内总行数-1)，这里的rank值是组内排名序号（rank函数返回值）
  - `cume_dist`，小于等于当前值的行数除以分区内总行数
  - `ntile(n)`，将有序分区的行分发到不同组中，返回每一行它所属的组的编号（从1开始）。（n为整数）
    - 用于取数据的百分比，比如返回前百分之二十的数据，可以分成5组，取第一组即可

参考：
- [Hive 窗口函数、分析函数](https://www.cnblogs.com/skyEva/p/5730531.html)
- [Hive常用函数大全（二）（窗口函数、分析函数、增强group）](https://blog.csdn.net/scgaliguodong123_/article/details/60135385)
## 自定义函数
自定义函数包含三种UDF（一进一出）、UDAF（自定义聚合函数，多进一出）、UDTF（自定义表函数，一进多出）
> <https://cwiki.apache.org/confluence/display/Hive/HivePlugins>

### UDF（map阶段）
- 继承 `org.apache.hadoop.hive.ql.UDF` 类
- 实现UDF类的evaluate函数，就是这个函数实现你的业务逻辑。其次，evaluate函数支持重载，所以你一个类里可以有多个重载的包含不同业务逻辑的evaluate函数，调用时通过参数个数、类型、返回值来区分
  - 函数必须有返回值，哪怕是null
  - 打jar包
- 在hive命令行创建临时函数
  - `add jar "jar_path";`，jar_path是上面打的jar包位置
  - `create temporary function function_name as class_name;`，function_name是定义的函数名（不一定和jar包中那个函数同名），class_NAME是上面那个函数的权限类名（就是包名）


### UDAF（reduce阶段）
参考：
- [Hive UDAF开发--个人补充理解](https://blog.csdn.net/w124374860/article/details/81021474)
- [Hive自定义函数(UDF、UDAF)](https://blog.csdn.net/scgaliguodong123_/article/details/46993005)

这里顺便补充一下[spark中定义hive的udf、udaf](https://blog.csdn.net/bb23417274/article/details/87976136)

## 命令
- 修复表：`msck repair table table_name`，有时像直接将数据上传到仓库的中作为一个分区，但没有相应元数据信息，可以修复表添加相应元数据信息
