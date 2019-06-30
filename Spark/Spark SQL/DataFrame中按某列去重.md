# DataFrame中按某列去重

- 首先想到的是DataFrame的distinct函数，但不行，它是对所有列去重
- 其次想到了DataFrame的dropDuplicates函数，`val df1 = df.dropDuplicates(col1_name,col2_name)`
- 还有就是通过窗口函数row_number，按照去重列字段分区，选row_number返回值为1的row
```sql
select col1,col2
from (
  select *,row_number() over(partition by col1,col2 order by xxx) num from tempTable
)t
where t.num = 1;
```