
---
title: "JSON schema inference  with Apache Spark"
tags:
- Apache Spark
- PySpark
---

## title breakdown

- JSON: [JavaScript Object Notation](https://www.json.org/json-en.html), a popular [semi-structured data](https://en.wikipedia.org/wiki/Semi-structured_data) interchange format 
- schema inference: the process of using a computer to derive the emergent structure of semi-structured
- [Apache Spark](https://spark.apache.org/): a popular [[Open Source]] query execution engine

## introduction

Ingesting and bringing structure to semi-structured data is an extremely common data-related task. 

Traditionally, a large part of developing this structure was to define it manually ([example](https://sparkbyexamples.com/pyspark/pyspark-structtype-and-structfield/)), which can be extremely tedious and error prone. Luckily, we have tools that will take the data and infer the structure, or schema, from it. Namely, Apache Spark.

Inferring the schema of a lot of semi-structured data is one of Apache Spark's (under-appreciated) strengths. But almost all of the examples that showcase this powerful feature focus on file-based sources. What if I want to derive the schema of an in-memory semi-structured payload? What about embedded fields in otherwise structured data? What about ingesting Snowflake VARIANT types? **It turns out, you can do this with a barely-documented feature**. 

The intention of this post is to show how to infer the schema over a lot of JSON payloads. I'm sure that the same methods will work for other semi-structured formats like CSV/TSV/DSV, but these will not be shown in this post. The limiting factor is if the DataFrameReader supports inference across files for that format - [JSON](https://github.com/apache/spark/blob/master/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/json/JsonInferSchema.scala), [CSV](https://github.com/apache/spark/blob/master/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/csv/CSVInferSchema.scala) do already(linked only for reference), and I'm sure contributions for any other formats are accepted.

The structure of this post will be to show one way to apply structure to ingested JSON payloads by using `from_json`. Part of that will be showcasing that, even as of Spark 3.3.0, you need to supply a schema to `from_json`. Then, a couple of methods will be shown for deriving that schema from raw data, which frees you from creating it manually.


## from_json

Let's say we have some data with two columns, an integer `id` and JSON string `payload`:
(don't worry about understanding the details of this code, a vague understanding works)
```python
data = {"id": 1, "payload": """{"name": "christopher", "age": 420}"""}
df = spark.createDataFrame([data])
df.show()
```

Results in:
```python
+---+-----------------------------------+
|id |payload                            |
+---+-----------------------------------+
|1  |{"name": "christopher", "age": 420}|
+---+-----------------------------------+
```

This is great, we have our data. But there is one issue:
```python
>>> df.schema.simpleString()
'struct<id:bigint,payload:string>'
```

Payload is a leaf string field, not a struct. Ideally, name would be selectable. This doesn't work:
```python
>>> df.select("payload.name")
 pyspark.sql.utils.AnalysisException: Can't extract value from payload#53: need struct type but got string
```

To get to our desired state, we have a [couple of options](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.functions.get_json_object.html#pyspark.sql.functions.get_json_object) , but here's one: `from_json`
```python
import pyspark.sql.functions as F
schema = 'STRUCT<`age`: BIGINT, `name`: STRING>'
df = df.withColumn("payload", F.from_json("payload", schema))
```

```python
>>> df.select("payload.name").show()
+-----------+
|       name|
+-----------+
|christopher|
+-----------+
```

Now we're getting somewhere.

But we're missing one crucial thing: I've [drawn the rest of the owl](https://i.kym-cdn.com/photos/images/original/000/572/078/d6d.jpg). I snuck in `schema`. `from_json`, right now, requires a user-provided schema. 

The rest of this post will show two different ways to have Spark do this for you, and spoiler, one is better than the other. 

## alternative 1: schema_of_json

`schema_of_json` is a [native Spark function](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.functions.schema_of_json.html) that takes a JSON string literal and returns its schema.  Here's an example snippet:

```python
import pyspark.sql.functions as F
json_payload = """{"name": "christopher", "age":420}"""
schema = spark.range(1).select(F.schema_of_json(json_payload).alias("schema")).collect()[0]["schema"]
```

Results in:
```python
>>> schema                                                                      
'STRUCT<`age`: BIGINT, `name`: STRING>'
```

Excellent. But can we do better? What if I have a lot of rows and their schema is different? Can we derive a super schema that fits it all?

## alternative 2: our inference "hack"

If you take a look at the [documentation for DataFrameReader.json()](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.DataFrameReader.json.html#pyspark.sql.DataFrameReader.json), you'll notice a weird example at the bottom:

```python
>>> rdd = sc.textFile('python/test_support/sql/people.json')
>>> df2 = spark.read.json(rdd) 
>>> df2.dtypes 
[('age', 'bigint'), ('name', 'string')]
```

### explanation

At line 1, we create an RDD from a text file located at `python/test_support/sql/people.json` using a SparkContext `sc`. The RDD is composed of a single column of type string, one row per line of input.

Line 2 is the motivating part of this post. We create a DataFrame from the RDD by passing it to the SparkSession `spark`'s JSON reader.

Line 4 is the proof that we've inferred the schema, as line 4 is line 3's output and shows multiple columns and differing types (bigint for age), which is desirable when bringing structure to unstructured data.

### application

This seems promising. But the example doesn't give us what we really want - it's still loading from a file. Let's adapt it:

```python
import pyspark.sql.functions as F
json_payloads = ["""{"name": "Christopher"}""", """{"age": 420}"""]
rdd = sc.parallelize(json_payloads) # needs to be a list of strings
schema = spark.read.json(rdd).schema
```

Resuts in
```python
>>> schema.simpleString()
'struct<age:bigint,name:string>'
```

There we go, same schema as before, and it's the expected one - note that the other schema was uppercased and spaced differently - this doesn't matter. We took two separate lines of JSON, each with different schemas, and created a super schema that accurately describes them both. Although it's a little hacky, alternative 2 is categorically better than alternative 1. It's more robust as it can handle more than one row.

### crossing and dotting

You can do this with existing DataFrame data

```python
data = {"id": 1, "payload": """{"name": "christopher", "age": 420}"""}
df = spark.createDataFrame([data])
payload_rdd = df.select("payload").rdd.map(lambda row: row[0])
schema = spark.read.json(payload_rdd).schema
df = df.withColumn("payload", F.from_json("payload", schema))
```

A common application for this would be for exporting data from Snowflake. Snowflake offers a [VARIANT type](https://docs.snowflake.com/en/sql-reference/data-types-semistructured.html#variant) that is encoded as a JSON string upon export. Or maybe you have Parquet files with JSON string columns.

## difficulties with semi-structured data

Structure, as it relates to semi-structured data, is best described as a spectrum. There is _well_ structured semi-structured data, and there is _poorly_ structured semi-structured data, and data in-between.

The above solutions will not work well with _extremely poorly_ structured semi-structured data. What I mean by _extremely poorly_ structured here is if you have, lets say, 5K+ sparsely populated and variable fields. You can give it a try, but there are no guarantees that applying structure to such a monstrosity will work.

## FAQ

Q: What happens when I am missing a field in the schema and apply it to raw data with `from_json`?

A: You will lose that field. If this is something you're afraid of, here's what I suggest: keep the raw text and parsed structure. That way, if you miss a field, you can do a self-referential backfill on your table to correct the parsed structure. Otherwise, if you subscribe to the bronze/silver/gold methodology, and this parsing is not part of your bronze layer (and it shouldn't be), you can simply stream from the layer that comes before and do a backfill that way.

## conclusion

This post went over a couple of ways of inferring the schema of JSON payloads. You can apply these methods to lessen your development burden and avoid mistakes.

The desired terminal state of this post is obsolescence. Hopefully we as Spark developers  get native support for semi-structured schema inference soon. Or some kind of JSON shredding functionality.