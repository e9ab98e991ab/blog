#SQLite数据类型

##概述
我们熟知的数据库引擎大部分采用静态数据类型，即列定义的类型定义了值的存储，并且值要严格满足列的定义，同一列所有值的存储方式都相同，比如定义了一个列类型为整型 int，不能在该列上输入'abc'。SQLite的数据类型则采用了动态类型，列定义不能决定值的存储，值的存储由值本身决定，因此在SQLite中，同一列会有多种存储方式。 
##数据类型与存储类
![](https://github.com/wslaimin/blog/raw/master/pics/dataType.png)
在SQLite中，存储分类和数据类型不是完全等价的，如INTEGER存储类别可以包含6种不同长度的Integer数据类型，然而这些INTEGER数据一旦被读入到内存后，SQLite会将其全部视为占用8个字节有符号整型。因此对于SQLite而言，同一个字段类型，可以在该字段中存储不同类型的数据，而且即便值的存储类型相同，底层存储占用的空间也与值相关，比如有的INTEGER占用1个字节，有的INTEGER可能占用8个字节。

(1).布尔数据类型：
SQLite并没有提供专门的布尔存储类型，取而代之的是存储整型1表示true，0表示false。

(2).日期和时间数据类型：
和布尔类型一样，SQLite也同样没有提供专门的日期时间存储类型，而是以TEXT、REAL和INTEGER类型分别不同的格式表示该类型，如：
TEXT: "YYYY-MM-DD HH:MM:SS.SSS"
REAL: 以Julian日期格式存储(儒略日(julian date)是自公元前4713年1月1日中午12时起经过的天数)
INTEGER: 以Unix时间形式保存数据值，即从1970-01-01 00:00:00到当前时间所流经的秒数。
SQLite提供typeof函数，用户可以根据这个函数来确定给定值的存储类型。 

##类型亲缘性
为了最大化SQLite和其它数据库引擎之间的数据类型兼容性，SQLite提出了"类型亲缘性(Type Affinity)"的概念。我们可以这样理解"类型亲缘性 "，在表字段被声明之后，SQLite都会根据该字段声明时的类型为其选择一种亲缘类型，当数据插入时，该字段的数据将会优先采用亲缘类型作为该值的存储方式，除非亲缘类型不匹配或无法转换当前数据到该亲缘类型，这样SQLite才会考虑其它更适合该值的类型存储该值。SQLite目前的版本支持以下五种亲缘类型：
![](https://www.github.com/wslaimin/blog/raw/master/pics/affinity.png)　　
###字段亲缘性的规则
字段的亲缘性是根据该字段在声明时被定义的类型来决定的，具体的规则可以参照以下列表。需要注意的是以下列表的顺序，即如果某一字段类型同时符合两种亲缘性，那么排在前面的规则将先产生作用。

1). 如果类型字符串中包含"INT"，那么该字段的亲缘类型是INTEGER。
2). 如果类型字符串中包含"CHAR"、"CLOB"或"TEXT"，那么该字段的亲缘类型是TEXT，如VARCHAR。
3). 如果类型字符串中包含"BLOB"，那么该字段的亲缘类型是NONE。
4). 如果类型字符串中包含"REAL"、"FLOA"或"DOUB"，那么该字段的亲缘类型是REAL。
5). 其余情况下，字段的亲缘类型为NUMERIC。
###具体示例
![](https://www.github.com/wslaimin/blog/raw/master/pics/affinity_example.png)　
4.比较与排序
在SQLite3中支持的比较表达式有："=", "==", "<", "<=", ">", ">=", "!=", "<>", "IN", "NOT IN", "BETWEEN", "IS" and "IS NOT"。数据的比较结果主要依赖于操作数的存储方式，其规则为：
1). 存储方式为NULL的数值小于其它存储类型的值。
2). 存储方式为INTEGER和REAL的数值小于TEXT或BLOB类型的值，如果同为INTEGER或REAL，则基于数值规则进行比较。
3). 存储方式为TEXT的数值小于BLOB类型的值。
4). 如果是两个BLOB类型的数值进行比较，其结果为C运行时函数memcmp()的结果。
5). 如果同为TEXT，SQLite利用特定的比较规则来判断，支持3种比较规则：
![](https://www.github.com/wslaimin/blog/raw/master/pics/data_compare.png)　
通过建表语句在可以在指定列上指定校对规则，比如：

```
CREATE TABLE t1(
    x INTEGER PRIMARY KEY,
    a,                 /* collating sequence BINARY */
    b COLLATE BINARY,  /* collating sequence BINARY */
    c COLLATE RTRIM,   /* collating sequence RTRIM  */
    d COLLATE NOCASE   /* collating sequence NOCASE */
);
```

<a href="https://www.sqlite.org/datatype3.html">参考文档</a>