<!--

    Licensed to the Apache Software Foundation (ASF) under one
    or more contributor license agreements.  See the NOTICE file
    distributed with this work for additional information
    regarding copyright ownership.  The ASF licenses this file
    to you under the Apache License, Version 2.0 (the
    "License"); you may not use this file except in compliance
    with the License.  You may obtain a copy of the License at
    
        http://www.apache.org/licenses/LICENSE-2.0
    
    Unless required by applicable law or agreed to in writing,
    software distributed under the License is distributed on an
    "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
    KIND, either express or implied.  See the License for the
    specific language governing permissions and limitations
    under the License.

-->

# 选择表达式

`SELECT` 子句指定查询的输出，由若干个 `selectExpr` 组成。 每个 `selectExpr` 定义了查询结果中的一列或多列。

**`selectExpr` 是一个由时间序列路径后缀、常量、函数和运算符组成的表达式。即 `selectExpr` 中可以包含：**
- 时间序列路径后缀（支持使用通配符）
- 运算符
  - 算数运算符
  - 比较运算符
  - 逻辑运算符
- 函数
  - 聚合函数
  - 时间序列生成函数（包括内置函数和用户自定义函数）
- 常量

## 使用别名

由于 IoTDB 独特的数据模型，在每个传感器前都附带有设备等诸多额外信息。有时，我们只针对某个具体设备查询，而这些前缀信息频繁显示造成了冗余，影响了结果集的显示与分析。

IoTDB 支持使用`AS`为查询结果集中的列指定别名。

**示例：**

```sql
select s1 as temperature, s2 as speed from root.ln.wf01.wt01;
```

结果集将显示为：

| Time | temperature | speed |
| ---- | ----------- | ----- |
| ...  | ...         | ...   |

## 运算符

IoTDB 中支持的运算符列表见文档 [运算符和函数](../Operators-Functions/Overview.md)。

## 函数

### 聚合函数

聚合函数是多对一函数。它们对一组值进行聚合计算，得到单个聚合结果。

**包含聚合函数的查询称为聚合查询**，否则称为时间序列查询。

**注意：聚合查询和时间序列查询不能混合使用。** 下列语句是不支持的：

```sql
select s1, count(s1) from root.sg.d1;
select sin(s1), count(s1) from root.sg.d1;
select s1, count(s1) from root.sg.d1 group by ([10,100),10ms);
```

IoTDB 支持的聚合函数见文档 [聚合函数](../Operators-Functions/Aggregation.md)。

### 时间序列生成函数

时间序列生成函数接受若干原始时间序列作为输入，产生一列时间序列输出。与聚合函数不同的是，时间序列生成函数的结果集带有时间戳列。

所有的时间序列生成函数都可以接受 * 作为输入，都可以与原始时间序列查询混合进行。

#### 内置时间序列生成函数

IoTDB 中支持的内置函数列表见文档 [运算符和函数](../Operators-Functions/Overview.md)。

#### 自定义时间序列生成函数

IoTDB 支持支持通过用户自定义函数（点击查看： [用户自定义函数](../Operators-Functions/User-Defined-Function.md) ） 能力进行函数功能扩展。

## 嵌套表达式举例

IoTDB 支持嵌套表达式，由于聚合查询和时间序列查询不能在一条查询语句中同时出现，我们将支持的嵌套表达式分为时间序列查询嵌套表达式和聚合查询嵌套表达式两类。

### 时间序列查询嵌套表达式

IoTDB 支持在 `SELECT` 子句中计算由**时间序列、常量、时间序列生成函数（包括用户自定义函数）和运算符**组成的任意嵌套表达式。

**说明：**

- 当某个时间戳下左操作数和右操作数都不为空（`null`）时，表达式才会有结果，否则表达式值为`null`，且默认不出现在结果集中。 
- 如果表达式中某个操作数对应多条时间序列（如通配符 `*`），那么每条时间序列对应的结果都会出现在结果集中（按照笛卡尔积形式）。

**示例 1：**

```sql
select a,
       b,
       ((a + 1) * 2 - 1) % 2 + 1.5,
       sin(a + sin(a + sin(b))),
       -(a + b) * (sin(a + b) * sin(a + b) + cos(a + b) * cos(a + b)) + 1
from root.sg1;
```

运行结果：

```
+-----------------------------+----------+----------+----------------------------------------+---------------------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------+
|                         Time|root.sg1.a|root.sg1.b|((((root.sg1.a + 1) * 2) - 1) % 2) + 1.5|sin(root.sg1.a + sin(root.sg1.a + sin(root.sg1.b)))|(-root.sg1.a + root.sg1.b * ((sin(root.sg1.a + root.sg1.b) * sin(root.sg1.a + root.sg1.b)) + (cos(root.sg1.a + root.sg1.b) * cos(root.sg1.a + root.sg1.b)))) + 1|
+-----------------------------+----------+----------+----------------------------------------+---------------------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------+
|1970-01-01T08:00:00.010+08:00|         1|         1|                                     2.5|                                 0.9238430524420609|                                                                                                                      -1.0|
|1970-01-01T08:00:00.020+08:00|         2|         2|                                     2.5|                                 0.7903505371876317|                                                                                                                      -3.0|
|1970-01-01T08:00:00.030+08:00|         3|         3|                                     2.5|                                0.14065207680386618|                                                                                                                      -5.0|
|1970-01-01T08:00:00.040+08:00|         4|      null|                                     2.5|                                               null|                                                                                                                      null|
|1970-01-01T08:00:00.050+08:00|      null|         5|                                    null|                                               null|                                                                                                                      null|
|1970-01-01T08:00:00.060+08:00|         6|         6|                                     2.5|                                -0.7288037411970916|                                                                                                                     -11.0|
+-----------------------------+----------+----------+----------------------------------------+---------------------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------+
Total line number = 6
It costs 0.048s
```

**示例 2：**

```sql
select (a + b) * 2 + sin(a) from root.sg
```

运行结果：

```
+-----------------------------+----------------------------------------------+
|                         Time|((root.sg.a + root.sg.b) * 2) + sin(root.sg.a)|
+-----------------------------+----------------------------------------------+
|1970-01-01T08:00:00.010+08:00|                             59.45597888911063|
|1970-01-01T08:00:00.020+08:00|                            100.91294525072763|
|1970-01-01T08:00:00.030+08:00|                            139.01196837590714|
|1970-01-01T08:00:00.040+08:00|                            180.74511316047935|
|1970-01-01T08:00:00.050+08:00|                            219.73762514629607|
|1970-01-01T08:00:00.060+08:00|                             259.6951893788978|
|1970-01-01T08:00:00.070+08:00|                             300.7738906815579|
|1970-01-01T08:00:00.090+08:00|                             39.45597888911063|
|1970-01-01T08:00:00.100+08:00|                             39.45597888911063|
+-----------------------------+----------------------------------------------+
Total line number = 9
It costs 0.011s
```

**示例 3：**

```sql
select (a + *) / 2  from root.sg1
```

运行结果：

```
+-----------------------------+-----------------------------+-----------------------------+
|                         Time|(root.sg1.a + root.sg1.a) / 2|(root.sg1.a + root.sg1.b) / 2|
+-----------------------------+-----------------------------+-----------------------------+
|1970-01-01T08:00:00.010+08:00|                          1.0|                          1.0|
|1970-01-01T08:00:00.020+08:00|                          2.0|                          2.0|
|1970-01-01T08:00:00.030+08:00|                          3.0|                          3.0|
|1970-01-01T08:00:00.040+08:00|                          4.0|                         null|
|1970-01-01T08:00:00.060+08:00|                          6.0|                          6.0|
+-----------------------------+-----------------------------+-----------------------------+
Total line number = 5
It costs 0.011s
```

**示例 4：**

```sql
select (a + b) * 3 from root.sg, root.ln
```

运行结果：

```
+-----------------------------+---------------------------+---------------------------+---------------------------+---------------------------+
|                         Time|(root.sg.a + root.sg.b) * 3|(root.sg.a + root.ln.b) * 3|(root.ln.a + root.sg.b) * 3|(root.ln.a + root.ln.b) * 3|
+-----------------------------+---------------------------+---------------------------+---------------------------+---------------------------+
|1970-01-01T08:00:00.010+08:00|                       90.0|                      270.0|                      360.0|                      540.0|
|1970-01-01T08:00:00.020+08:00|                      150.0|                      330.0|                      690.0|                      870.0|
|1970-01-01T08:00:00.030+08:00|                      210.0|                      450.0|                      570.0|                      810.0|
|1970-01-01T08:00:00.040+08:00|                      270.0|                      240.0|                      690.0|                      660.0|
|1970-01-01T08:00:00.050+08:00|                      330.0|                       null|                       null|                       null|
|1970-01-01T08:00:00.060+08:00|                      390.0|                       null|                       null|                       null|
|1970-01-01T08:00:00.070+08:00|                      450.0|                       null|                       null|                       null|
|1970-01-01T08:00:00.090+08:00|                       60.0|                       null|                       null|                       null|
|1970-01-01T08:00:00.100+08:00|                       60.0|                       null|                       null|                       null|
+-----------------------------+---------------------------+---------------------------+---------------------------+---------------------------+
Total line number = 9
It costs 0.014s
```

### 聚合查询嵌套表达式

IoTDB 支持在 `SELECT` 子句中计算由**聚合函数、常量、时间序列生成函数和表达式**组成的任意嵌套表达式。

**说明：**
- 当某个时间戳下左操作数和右操作数都不为空（`null`）时，表达式才会有结果，否则表达式值为`null`，且默认不出现在结果集中。但在使用`GROUP BY`子句的聚合查询嵌套表达式中，我们希望保留每个时间窗口的值，所以表达式值为`null`的窗口也包含在结果集中。
- 如果表达式中某个操作数对应多条时间序列（如通配符`*`），那么每条时间序列对应的结果都会出现在结果集中（按照笛卡尔积形式）。

**示例 1：**

```sql
select avg(temperature),
       sin(avg(temperature)),
       avg(temperature) + 1,
       -sum(hardware),
       avg(temperature) + sum(hardware)
from root.ln.wf01.wt01;
```

运行结果：

```
+----------------------------------+---------------------------------------+--------------------------------------+--------------------------------+--------------------------------------------------------------------+
|avg(root.ln.wf01.wt01.temperature)|sin(avg(root.ln.wf01.wt01.temperature))|avg(root.ln.wf01.wt01.temperature) + 1|-sum(root.ln.wf01.wt01.hardware)|avg(root.ln.wf01.wt01.temperature) + sum(root.ln.wf01.wt01.hardware)|
+----------------------------------+---------------------------------------+--------------------------------------+--------------------------------+--------------------------------------------------------------------+
|                15.927999999999999|                   -0.21826546964855045|                    16.927999999999997|                         -7426.0|                                                            7441.928|
+----------------------------------+---------------------------------------+--------------------------------------+--------------------------------+--------------------------------------------------------------------+
Total line number = 1
It costs 0.009s
```

**示例 2：**

```sql
select avg(*), 
	   (avg(*) + 1) * 3 / 2 -1 
from root.sg1
```

运行结果：

```
+---------------+---------------+-------------------------------------+-------------------------------------+
|avg(root.sg1.a)|avg(root.sg1.b)|(avg(root.sg1.a) + 1) * 3 / 2 - 1    |(avg(root.sg1.b) + 1) * 3 / 2 - 1    |
+---------------+---------------+-------------------------------------+-------------------------------------+
|            3.2|            3.4|                    5.300000000000001|                   5.6000000000000005|
+---------------+---------------+-------------------------------------+-------------------------------------+
Total line number = 1
It costs 0.007s
```

**示例 3：**

```sql
select avg(temperature),
       sin(avg(temperature)),
       avg(temperature) + 1,
       -sum(hardware),
       avg(temperature) + sum(hardware) as custom_sum
from root.ln.wf01.wt01
GROUP BY([10, 90), 10ms);
```

运行结果：

```
+-----------------------------+----------------------------------+---------------------------------------+--------------------------------------+--------------------------------+----------+
|                         Time|avg(root.ln.wf01.wt01.temperature)|sin(avg(root.ln.wf01.wt01.temperature))|avg(root.ln.wf01.wt01.temperature) + 1|-sum(root.ln.wf01.wt01.hardware)|custom_sum|
+-----------------------------+----------------------------------+---------------------------------------+--------------------------------------+--------------------------------+----------+
|1970-01-01T08:00:00.010+08:00|                13.987499999999999|                     0.9888207947857667|                    14.987499999999999|                         -3211.0| 3224.9875|
|1970-01-01T08:00:00.020+08:00|                              29.6|                    -0.9701057337071853|                                  30.6|                         -3720.0|    3749.6|
|1970-01-01T08:00:00.030+08:00|                              null|                                   null|                                  null|                            null|      null|
|1970-01-01T08:00:00.040+08:00|                              null|                                   null|                                  null|                            null|      null|
|1970-01-01T08:00:00.050+08:00|                              null|                                   null|                                  null|                            null|      null|
|1970-01-01T08:00:00.060+08:00|                              null|                                   null|                                  null|                            null|      null|
|1970-01-01T08:00:00.070+08:00|                              null|                                   null|                                  null|                            null|      null|
|1970-01-01T08:00:00.080+08:00|                              null|                                   null|                                  null|                            null|      null|
+-----------------------------+----------------------------------+---------------------------------------+--------------------------------------+--------------------------------+----------+
Total line number = 8
It costs 0.012s
```