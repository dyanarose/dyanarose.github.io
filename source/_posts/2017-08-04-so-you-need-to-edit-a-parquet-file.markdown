---
layout: post
title: "so you need to edit a parquet file"
date: 2017-08-04 11:40:32 +0100
comments: false
categories:
published: true
---

You've uncovered a problem in your beautiful parquet files, some piece of data either snuck in, or was calculated incorrectly, or there was just a bug. You know exactly how to correct the data, but how do you update the files?

tl;dr: [spark-edit-examples](https://github.com/dyanarose/parquet-edit-examples)

### It's all immutable

The problem we have when we need to edit the data is that our data structures are immutable.

You can add partitions to Parquet files, but you can't edit the data in place. Spark DataFrames are immutable.

But ultimately we can mutate the data, we just need to accept that we won't be doing it in place. We will need to recreate the Parquet files using a combination of schemas and UDFs to correct the bad data.

## Schemas

Reading in data using a schema gives you a lot of power over the resultant structure of the DataFrame (not to mention it makes reading in json files a lot faster, and will allow you to union compatible Parquet files)

#### Case 1: I need to drop an entire column
To drop an entire column, read the data in with a schema that doesn't contain that column. When you write the DataFrame back out, the column will no longer exist

[ColumnTransform.scala](https://github.com/dyanarose/parquet-edit-examples/blob/master/transform-examples/src/main/scala/com/dlr/transform/transformers/ColumnTransform.scala)

```scala
object ColumnTransform {

  def transform(spark: SparkSession, sourcePath: String, destPath: String): Unit = {

    // read in the data with a new schema
    val allGoodData = spark.read.schema(ColumnDropSchema.schema).parquet(sourcePath)

    // write out the final edited data
    allGoodData.write.parquet(destPath)
  }
}
```

#### Case 2: I need to drop full rows of data
To drop full rows, read in the data and select the data you want to save into a new DataFrame using a where clause. When you write the new DataFrame it will only have the rows that match the where clause.

[WhereTransform.scala](https://github.com/dyanarose/parquet-edit-examples/blob/master/transform-examples/src/main/scala/com/dlr/transform/transformers/WhereTransform.scala)
```scala
object WhereTransform {

  def transform(spark: SparkSession, sourcePath: String, destPath: String): Unit = {
    val originalData = spark.read.schema(RawDataSchema.schema).parquet(sourcePath)

    // select only the good data rows
    val allGoodData = originalData.where("myField is null")

    // write out the final edited data
    allGoodData.write.parquet(destPath)
  }
}
```

## User Defined Functions (UDFs)

[UDFs in Spark](https://blog.cloudera.com/blog/2017/02/working-with-udfs-in-apache-spark/) are used to apply functions to a row of data. The result of the UDF becomes the field value.

Note that when using UDFs you must alias the resultant column otherwise it will end up renamed similar to `UDF(fieldName)`

#### Case 3: I need to edit the value of a simple type (String, Boolean, ...)
To edit a simple type you first need to create a function that takes and returns the same type.

This function is then registered for use as a UDF and it can then be applied to a field in a select clause


[SimpleTransform.scala](https://github.com/dyanarose/parquet-edit-examples/blob/master/transform-examples/src/main/scala/com/dlr/transform/transformers/SimpleTransform.scala)
```scala
object SimpleTransform {

  def transform(spark: SparkSession, sourcePath: String, destPath: String): Unit = {
    val originalData = spark.read.schema(RawDataSchema.schema).parquet(sourcePath)

    // take in a String, return a String
    // cleanFunc takes the String field value and return the empty string in its place
    // you can interrogate the value and return any String here
    def cleanFunc: (String => String) = { _ => "" }

    // register the func as a udf
    val clean = udf(cleanFunc)

    // required for the $ column syntax
    import spark.sqlContext.implicits._

    // if you have data that doesn't need editing, you can separate it out
    // The data will need to be in a form that can be unioned with the edited data
    // That can be done by selecting out the fields in the same way in both the good and transformed data sets.
    val alreadyGoodData = originalData.where("myField is null").select(
      Seq[Column](
        $"myField",
        $"myMap",
        $"myStruct"
      ):_*
    )

    // apply the udf to the fields that need editing
    // selecting out all the data that will be present in the final parquet file
    val transformedData = originalData.where("myField is not null").select(
      Seq[Column](
        clean($"myField").as("myField"),
        $"myMap",
        $"myStruct"
      ):_*
    )

    // union the two DataFrames
    val allGoodData = alreadyGoodData.union(transformedData)

    // write out the final edited data
    allGoodData.write.parquet(destPath)
  }
}
```

#### Case 4: I need to edit the value of a MapType

MapTypes follow the same pattern as simple types. You write a function that takes a Map of the correct key and value types and returns a Map of the same types.

In the following example, an entire entry in the Map[String,String] is removed from the final data by filtering on the keyset.

[MapTypeTransform.scala](https://github.com/dyanarose/parquet-edit-examples/blob/master/transform-examples/src/main/scala/com/dlr/transform/transformers/MapTypeTransform.scala)
```scala
object MapTypeTransform {

  def transform(spark: SparkSession, sourcePath: String, destPath: String): Unit = {
    val originalData = spark.read.schema(RawDataSchema.schema).parquet(sourcePath)

    // cleanFunc will simply take the MapType and return an edited Map
    // in this example it removes one member of the map before returning
    def cleanFunc: (Map[String, String] => Map[String, String]) = { m => m.filterKeys(k => k != "editMe") }

    // register the func as a udf
    val clean = udf(cleanFunc)

    // required for the $ column syntax
    import spark.sqlContext.implicits._

    // if you have data that doesn't need editing, you can separate it out
    // The data will need to be in a form that can be unioned with the edited data
    // I do that here by selecting out all the fields.
    val alreadyGoodData = originalData.where("myMap.editMe is null").select(
      Seq[Column](
        $"myField",
        $"myMap",
        $"myStruct"
      ):_*
    )

    // apply the udf to the fields that need editing
    // selecting out all the data that will be present in the final parquet file
    val transformedData = originalData.where("myMap.editMe is not null").select(
      Seq[Column](
        $"myField",
        clean($"myMap").as("myMap"),
        $"myStruct"
      ):_*
    )

    // union the two DataFrames
    val allGoodData = alreadyGoodData.union(transformedData)

    // write out the final edited data
    allGoodData.write.parquet(destPath)
  }
}
```

#### Case 5: I need to change the value of a member of a StructType

Working with StructTypes requires an addition to the UDF registration statement. By supplying the schema of the StructType you are able to manipulate using a function that takes and returns a Row. 

As Rows are immutable, a new Row must be created that has the same field order, type, and number as the schema. But, since the schema of the data is known, it's relatively easy to reconstruct a new Row with the correct fields.

[StructTypeTransform.scala](https://github.com/dyanarose/parquet-edit-examples/blob/master/transform-examples/src/main/scala/com/dlr/transform/transformers/StructTypeTransform.scala)
```scala
object StructTypeTransform {

  def transform(spark: SparkSession, sourcePath: String, destPath: String): Unit = {
    val originalData = spark.read.schema(RawDataSchema.schema).parquet(sourcePath)

    // cleanFunc will take the struct as a Row and return a new Row with edited fields
    // note that the ordering and count of the fields must remain the same
    def cleanFunc: (Row => Row) = { r => RowFactory.create(r.getAs[BooleanType](0), "") }

    // register the func as a udf
    // give the UDF a schema or the Row type won't be supported
    val clean = udf(cleanFunc,
      StructType(
        StructField("myField", BooleanType, true) ::
          StructField("editMe", StringType, true) ::
          Nil
      )
    )

    // required for the $ column syntax
    import spark.sqlContext.implicits._

    // if you have data that doesn't need editing, you can separate it out
    // The data will need to be in a form that can be unioned with the edited data
    // I do that here by selecting out all the fields.
    val alreadyGoodData = originalData.where("myStruct.editMe is null").select(
      Seq[Column](
        $"myField",
        $"myStruct",
        $"myMap"
      ):_*
    )

    // apply the udf to the fields that need editing
    // selecting out all the data that will be present in the final parquet file
    val transformedData = originalData.where("myStruct.editMe is not null").select(
      Seq[Column](
        $"myField",
        clean($"myStruct").as("myStruct"),
        $"myMap"
      ):_*
    )

    // union the two DataFrames
    val allGoodData = alreadyGoodData.union(transformedData)

    // write out the final edited data
    allGoodData.write.parquet(destPath)
  }
}
```

### Finally

Always test your transforms before you delete the original data!
