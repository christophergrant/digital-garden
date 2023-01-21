---
title: "Apache Spark"
---

**Disclaimer** -- I currently work at Databricks, the company that is home to many of the original creators of Apache Spark. Even more than that, I currently work in Sales at Databricks. 

The opinions expressed on this page and on this blog as a whole do not express the opinions of Databricks. And although I'd like to think of myself as neutral, my experience at Databricks has forged my opinions. Interpret this however you'd like.

## Tips for learning Apache Spark in 202x

### Use PySpark

Spark has many dialects including Scala, Python, SQL, R, and other third-party bindings. As it stands right now, Python and SQL are _the_ end-user languages for data. 

You can interoperate between Python and SQL by using PySpark's [spark.sql method](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.SparkSession.sql.html), e.g:
```python
df = spark.sql("SELECT * FROM table")
df.write.format("delta").save("/your/path/here")
```
or even something like:
```python
spark.sql("""CREATE TABLE table AS 
		  SELECT * FROM other_table""")
```

So you get that base level functionality and familiarity of SQL and the added imperative flexibilty of Python.

I personally skew towards the DataFrame API for data engineering tasks because of the traumatic experiences I have had with massive piles with SQL and YAML. But the point is, you can execute a beautiful hybrid model that is easily taught to those familiar with SQL.

Traditionally, many have questioned whether or not you need to be a Scala guru to use Spark. A spicy take here: there are essentially no reasons to use the Scala dialect nowadays, due to factors like
- The performance differences between Scala Spark and other Spark dialects has been minimal for a long time. 
- Python is much more accessible to end users than Scala is.
- Scala's strongly-typed DataSet API never really took off.

### Do not start with RDDs, use DataFrames/SQL instead

RDDs are the foundational and low-level Spark abstraction. DataFrames/SQL are higher level and carry a lot of benefits on top of RDDs. 

For these reasons among others, new Spark applications should use the newer DataFrame/SQL APIs and legacy applications should be rewritten.

This carries over to other Spark APIs, on the left are the older, lower-level APIs, where the right has the newer APIs:
- Spark core (RDDs) => Spark SQL and DataFrames
- Spark Streaming => Spark Structured Streaming
- Spark MLLib => SparkML
- GraphX => GraphFrames

### Use a modern table format

Table formats like Linux Foundation [Delta Lake](https://docs.delta.io/latest/index.html_) and [Apache Iceberg](https://iceberg.apache.org) have changed the data lake game. You get all of the benefits of columnar/binary file formats like Parquet, ORC, Avro, but with the added benefits of:
- Convinient user APIs for Upserts/Deletes
- ACID properties
- Richer schema enforcement capabilities
- and much more

### Use hydro


## Spark Data Sources

**Note: these listings include third-party libraries with varying SDLC standards, use these at your own risk**

**This list is not exhaustive. Please feel free to open a PR against this page to add/remove connectors**

| Connector      | Reference |
| ----------- | ----------- |
| CSV/TSV/DSV   |  https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.DataFrameReader.csv.html       |
| JSON   | https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.DataFrameReader.json.html#pyspark.sql.DataFrameReader.json        |
| XML   | https://github.com/databricks/spark-xml        |
| Text   | https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.DataFrameReader.text.html#pyspark.sql.DataFrameReader.text        |
| Binary   | https://spark.apache.org/docs/latest/sql-data-sources-binaryFile.html        |
| Avro   | https://spark.apache.org/docs/latest/sql-data-sources-avro.html        |
| ORC   | https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.DataFrameReader.orc.html#pyspark.sql.DataFrameReader.orc        |
| Parquet      | https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.DataFrameReader.parquet.html#pyspark.sql.DataFrameReader.parquet       |
| Kafka   | https://spark.apache.org/docs/3.3.1/structured-streaming-kafka-integration.html#content        |
| Amazon Kinesis   | https://docs.databricks.com/structured-streaming/kinesis.html        |
| Azure EventHubs   | https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-kafka-spark-tutorial        |
| Google Pub/Sub   | Databricks (In preview)        |
| Delta Lake   | https://delta.io/        |
| Iceberg   | https://iceberg.apache.org        |
| Hudi   | https://hudi.apache.org/        |
| JDBC   | https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.DataFrameReader.jdbc.html?highlight=jdbc#pyspark.sql.DataFrameReader.jdbc        |
| MongoDB   | https://www.mongodb.com/docs/spark-connector/current/        |
| Snowflake   | https://docs.snowflake.com/en/user-guide/spark-connector.html        |
| Google BigQuery   | https://github.com/GoogleCloudDataproc/spark-bigquery-connector        |
| Amazon Redshift   | https://docs.databricks.com/external-data/amazon-redshift.html        |


## Respected Spark-Related Content Creators

[Holden Karau](http://holdenkarau.com/) - Spark committer, conference talks

[Jacek Laskowski](https://github.com/jaceklaskowski) - GitBooks pages on Apache Spark

[Bartosz Konieczny](https://www.waitingforcode.com/) - WaitingForCode blog

[Nick Karpov](https://www.linkedin.com/in/nick-karpov/) - LinkedIn posts, Delta Users Slack

[Matthew Powers](https://github.com/MrPowers) - Blog posts, various helper libraries

