---
layout: post
title: "Understanding the different Sparl SQL DataTypes"
date: 2016-04-09 12:57:06 +0100
comments: false
categories: [Apache Spark, Spark SQL]
published: true
---

Exploring how different DataTypes in Spark SQL are imported from line delimited json. 

The only one I really can't get working yet is the CalendarIntervalType.  

Looking at the Spark source files [literals.scala](https://github.com/apache/spark/blob/master/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/expressions/literals.scala) and [CalendarInterval.java](https://github.com/apache/spark/blob/master/common/unsafe/src/main/java/org/apache/spark/unsafe/types/CalendarInterval.java) I would assume that `CalendarInterval.fromString` is called with the value, however I'm just getting nulls back currently when passing in a value like 'interval 2 days' which when passed to `CalendarInterval.fromString` returns a non-null `CalendarInterval`.

Results:
```
-------------- DecimalType --------------
------- DecimalType Input
{'decimal': 1.2345}
{'decimal': 1}
{'decimal': 234.231}
{'decimal': Infinity}
{'decimal': -Infinity}
{'decimal': NaN}
{'decimal': '1'}
{'decimal': '1.2345'}
{'decimal': null}

------- DecimalType Inferred Schema
root
 |-- decimal: string (nullable = true)

+-----------+
|    decimal|
+-----------+
|     1.2345|
|          1|
|    234.231|
| "Infinity"|
|"-Infinity"|
|      "NaN"|
|          1|
|     1.2345|
|       null|
+-----------+


------- DecimalType Set Schema
root
 |-- decimal: decimal(6,3) (nullable = true)

+-------+
|decimal|
+-------+
|  1.235|
|  1.000|
|234.231|
|   null|
|   null|
|   null|
|   null|
|   null|
|   null|
+-------+



-------------- BooleanType --------------
------- BooleanType Input
{'boolean': true}
{'boolean': false}
{'boolean': 'false'}
{'boolean': 'true'}
{'boolean': null}
{'boolean': 1}
{'boolean': 0}
{'boolean': '1'}
{'boolean': '0'}
{'boolean': 'a'}

------- BooleanType Inferred Schema
root
 |-- boolean: string (nullable = true)

+-------+
|boolean|
+-------+
|   true|
|  false|
|  false|
|   true|
|   null|
|      1|
|      0|
|      1|
|      0|
|      a|
+-------+


------- BooleanType Set Schema
root
 |-- boolean: boolean (nullable = true)

+-------+
|boolean|
+-------+
|   true|
|  false|
|   null|
|   null|
|   null|
|   null|
|   null|
|   null|
|   null|
|   null|
+-------+



-------------- ByteType --------------
------- ByteType Input
{'byte': 'a'}
{'byte': 'b'}
{'byte': 1}
{'byte': 0}
{'byte': 5}
{'byte': null}

------- ByteType Inferred Schema
root
 |-- byte: string (nullable = true)

+----+
|byte|
+----+
|   a|
|   b|
|   1|
|   0|
|   5|
|null|
+----+


------- ByteType Set Schema
root
 |-- byte: byte (nullable = true)

+----+
|byte|
+----+
|null|
|null|
|   1|
|   0|
|   5|
|null|
+----+



-------------- CalendarIntervalType --------------
------- CalendarIntervalType Input
{'calendarInterval': 'interval 2 days'}
{'calendarInterval': 'interval 1 week'}
{'calendarInterval': 'interval 5 years'}
{'calendarInterval': 'interval 6 months'}
{'calendarInterval': 10}
{'calendarInterval': 'interval a'}
{'calendarInterval': null}

------- CalendarIntervalType Inferred Schema
root
 |-- calendarInterval: string (nullable = true)

+-----------------+
| calendarInterval|
+-----------------+
|  interval 2 days|
|  interval 1 week|
| interval 5 years|
|interval 6 months|
|               10|
|       interval a|
|             null|
+-----------------+


------- CalendarIntervalType Set Schema
root
 |-- calendarInterval: calendarinterval (nullable = true)

+----------------+
|calendarInterval|
+----------------+
|            null|
|            null|
|            null|
|            null|
|            null|
|            null|
|            null|
+----------------+



-------------- DateType --------------
------- DateType Input
{'date': '2016-04-24'}
{'date': '0001-01-01'}
{'date': '9999-12-31'}
{'date': '2016-04-24 12:10:01'}
{'date': 1461496201000}
{'date': null}

------- DateType Inferred Schema
root
 |-- date: string (nullable = true)

+-------------------+
|               date|
+-------------------+
|         2016-04-24|
|         0001-01-01|
|         9999-12-31|
|2016-04-24 12:10:01|
|      1461496201000|
|               null|
+-------------------+


------- DateType Set Schema
root
 |-- date: date (nullable = true)

+----------+
|      date|
+----------+
|2016-04-24|
|0001-01-01|
|9999-12-31|
|2016-04-24|
|      null|
|      null|
+----------+



-------------- DoubleType --------------
------- DoubleType Input
{'double': 1.23456}
{'double': 1}
{'double': 1.7976931348623157E308}
{'double': -1.7976931348623157E308}
{'double': Infinity}
{'double': -Infinity}
{'double': NaN}
{'double': '1'}
{'double': '1.23456'}
{'double': null}

------- DoubleType Inferred Schema
root
 |-- double: string (nullable = true)

+--------------------+
|              double|
+--------------------+
|             1.23456|
|                   1|
|1.797693134862315...|
|-1.79769313486231...|
|          "Infinity"|
|         "-Infinity"|
|               "NaN"|
|                   1|
|             1.23456|
|                null|
+--------------------+


------- DoubleType Set Schema
root
 |-- double: double (nullable = true)

+--------------------+
|              double|
+--------------------+
|             1.23456|
|                 1.0|
|1.797693134862315...|
|-1.79769313486231...|
|            Infinity|
|           -Infinity|
|                 NaN|
|                null|
|                null|
|                null|
+--------------------+



-------------- FloatType --------------
------- FloatType Input
{'float': 1.23456}
{'float': 1}
{'float': 3.4028235E38}
{'float': -3.4028235E38}
{'float': Infinity}
{'float': -Infinity}
{'float': NaN}
{'float': '1'}
{'float': '1.23456'}
{'float': null}

------- FloatType Inferred Schema
root
 |-- float: string (nullable = true)

+-------------+
|        float|
+-------------+
|      1.23456|
|            1|
| 3.4028235E38|
|-3.4028235E38|
|   "Infinity"|
|  "-Infinity"|
|        "NaN"|
|            1|
|      1.23456|
|         null|
+-------------+


------- FloatType Set Schema
root
 |-- float: float (nullable = true)

+-------------+
|        float|
+-------------+
|      1.23456|
|          1.0|
| 3.4028235E38|
|-3.4028235E38|
|     Infinity|
|    -Infinity|
|          NaN|
|         null|
|         null|
|         null|
+-------------+



-------------- IntegerType --------------
------- IntegerType Input
{'integer': 1}
{'integer': 2147483647}
{'integer': -2147483648}
{'integer': 2147483648}
{'integer': '1'}
{'integer': 1.23456}
{'integer': '1.23456'}
{'integer': null}

------- IntegerType Inferred Schema
root
 |-- integer: string (nullable = true)

+-----------+
|    integer|
+-----------+
|          1|
| 2147483647|
|-2147483648|
| 2147483648|
|          1|
|    1.23456|
|    1.23456|
|       null|
+-----------+


------- IntegerType Set Schema
root
 |-- integer: integer (nullable = true)

+-----------+
|    integer|
+-----------+
|          1|
| 2147483647|
|-2147483648|
|       null|
|       null|
|       null|
|       null|
|       null|
+-----------+



-------------- LongType --------------
------- LongType Input
{'long': 1}
{'long': 9223372036854775807}
{'long': -9223372036854775808}
{'long': '1'}
{'long': 1.23456}
{'long': '1.23456'}
{'long': null}

------- LongType Inferred Schema
root
 |-- long: string (nullable = true)

+--------------------+
|                long|
+--------------------+
|                   1|
| 9223372036854775807|
|-9223372036854775808|
|                   1|
|             1.23456|
|             1.23456|
|                null|
+--------------------+


------- LongType Set Schema
root
 |-- long: long (nullable = true)

+--------------------+
|                long|
+--------------------+
|                   1|
| 9223372036854775807|
|-9223372036854775808|
|                null|
|                null|
|                null|
|                null|
+--------------------+



-------------- MapType --------------
------- MapType Input
{'map': {'a_key': 'a value', 'b_key': 'b value'}}
{'map': {'key': 1, 'key1': null}}
{'map': null}

------- MapType Inferred Schema
root
 |-- map: struct (nullable = true)
 |    |-- a_key: string (nullable = true)
 |    |-- b_key: string (nullable = true)
 |    |-- key: long (nullable = true)
 |    |-- key1: string (nullable = true)

+--------------------+
|                 map|
+--------------------+
|[a value,b value,...|
|  [null,null,1,null]|
|                null|
+--------------------+


------- MapType Set Schema
root
 |-- map: map (nullable = true)
 |    |-- key: string
 |    |-- value: string (valueContainsNull = true)

+--------------------+
|                 map|
+--------------------+
|Map(a_key -> a va...|
|Map(key -> 1, key...|
|                null|
+--------------------+



-------------- NullType --------------
------- NullType Input
{'null': null}
{'null': true}
{'null': false}
{'null': 1}
{'null': 0}
{'null': '1'}
{'null': '0'}
{'null': 'a'}

------- NullType Inferred Schema
root
 |-- null: string (nullable = true)

+-----+
| null|
+-----+
| null|
| true|
|false|
|    1|
|    0|
|    1|
|    0|
|    a|
+-----+


------- NullType Set Schema
root
 |-- null: null (nullable = true)

+----+
|null|
+----+
|null|
|null|
|null|
|null|
|null|
|null|
|null|
|null|
+----+



-------------- ShortType --------------
------- ShortType Input
{'short': 0}
{'short': 1}
{'short': 32767}
{'short': -32768}
{'short': 32768}
{'short': 1.23456}
{'short': '0'}
{'short': '1'}
{'short': '1.23456'}
{'short': null}

------- ShortType Inferred Schema
root
 |-- short: string (nullable = true)

+-------+
|  short|
+-------+
|      0|
|      1|
|  32767|
| -32768|
|  32768|
|1.23456|
|      0|
|      1|
|1.23456|
|   null|
+-------+


------- ShortType Set Schema
root
 |-- short: short (nullable = true)

+------+
| short|
+------+
|     0|
|     1|
| 32767|
|-32768|
|  null|
|  null|
|  null|
|  null|
|  null|
|  null|
+------+



-------------- TimestampType --------------
------- TimestampType Input
{'timestamp': '2016-04-24'}
{'timestamp': '2016-04-24 12:10:01'}
{'timestamp': 1461496201000}
{'timestamp': '0001-01-01'}
{'timestamp': '9999-12-31'}
{'timestamp': null}

------- TimestampType Inferred Schema
root
 |-- timestamp: string (nullable = true)

+-------------------+
|          timestamp|
+-------------------+
|         2016-04-24|
|2016-04-24 12:10:01|
|      1461496201000|
|         0001-01-01|
|         9999-12-31|
|               null|
+-------------------+


------- TimestampType Set Schema
root
 |-- timestamp: timestamp (nullable = true)

+--------------------+
|           timestamp|
+--------------------+
|2016-04-24 00:00:...|
|2016-04-24 12:10:...|
|2016-04-24 12:10:...|
|0001-01-01 00:00:...|
|9999-12-31 00:00:...|
|                null|
+--------------------+




```