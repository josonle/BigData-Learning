# 将DataFrame中某一字符串列分割成多列

比如说某列由一个空格分隔的字符串组成
```scala
# var df = xxx
split_col = split(df.col("col_name"), " ")
df = df.withColumn("new_col1",split_col.getItem(0))
```

## 按某列中部分内容排序
```scala
# 比如某列元素类似2010-12-01 08:26:00
# 想按年月日排序
df.orderBy(split(df.col("Date"), " ").getItem(0)).show
```