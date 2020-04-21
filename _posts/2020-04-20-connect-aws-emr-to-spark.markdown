---
layout: post
title: "Connect AWS EMR to Redshift"
date: "2020-04-20 21:52:14 -0700"
---


I'm making this post since this Github page (https://github.com/databricks/spark-redshift) seems to be abandonded by Databricks. It's pretty good - so if you need details, that's a great place to start.

To do this, you need drivers for Spark to connect to Redshift. Download the following four files:

```
https://repo1.maven.org/maven2/com/databricks/spark-redshift_2.10/2.0.1/spark-redshift_2.10-2.0.1.jar
https://repo1.maven.org/maven2/com/eclipsesource/minimal-json/minimal-json/0.9.5/minimal-json-0.9.5.jar
https://repo1.maven.org/maven2/com/databricks/spark-avro_2.11/3.0.0/spark-avro_2.11-3.0.0.jar
https://s3.amazonaws.com/redshift-downloads/drivers/jdbc/1.2.41.1065/RedshiftJDBC4-no-awssdk-1.2.41.1065.jar
```

Next - you need to get these on your machine. If you're using EMR and have a notebook- you can do place the jars in s3 and reference then. Here, I stored them in my "myBucket" bucket

```

{ "conf": {"spark.jars":"s3://myBucket/spark-redshift_2.10-2.0.1.jar,"
                        "s3://myBucket/minimal-json-0.9.5.jar,"
                        "s3://myBucket/spark-avro_2.11-3.0.0.jar,"
                        "s3://myBucket/RedshiftJDBC4-no-awssdk-1.2.41.1065.jar"} }
```

This is a magic hint for the IPython notebook and will install your libraries at the beginning of the notebook.


Next - you can create a SQL query dataframe (this is Python, but the principle is the same in Scala/R):

```
from pyspark.sql import SQLContext
jdbcUrl = "jdbc:redshift://yourcluster.redshift.amazonaws.com:5439/dev?user=usernamehere&password=passwordhere"

sc = spark # existing SparkContext
sql_context = SQLContext(sc)

users_max_df = sql_context.read.format("com.databricks.spark.redshift")\
                                    .option("numPartitions",125)\
                                    .option("url", jdbcUrl)\
                                    .option("tempdir","s3://your-bucket/folder")\ #Used for COPY to S3
                                    .open()
                                    .option("query","select MAX(created_at) from users")\
                                    .load()

```

The above works if your EC2 instance has access to your S3 bucket through the IAM policy. If you don't have your EC2 instance being accessible to your S3 bucket - you can specify a different set of S3 credentials for the job.

```
from pyspark.sql import SQLContext
jdbcUrl = "jdbc:redshift://yourcluster.redshift.amazonaws.com:5439/dev?user=usernamehere&password=passwordhere"

sc = spark # existing SparkContext
sql_context = SQLContext(sc)

sql_context._jsc.hadoopConfiguration().set("fs.s3.awsAccessKeyId", "yourAccessKeyIDFromIAM")
sql_context._jsc.hadoopConfiguration().set("fs.s3.awsSecretAccessKey", "yourSecretAccessKeyFromIAM")

users_max_df = sql_context.read.format("com.databricks.spark.redshift")\
                                    .option("numPartitions",125)\
                                    .option("url", jdbcUrl)\
                                    .option("tempdir","s3://your-bucket/folder")\ #Used for COPY to S3
                                    .open()
                                    .option("forward_spark_s3_credentials",True)\
                                    .option("query","select MAX(created_at) from users")\
                                    .load()
```

Now you can have all your wonderful Redshift data in EMR.
