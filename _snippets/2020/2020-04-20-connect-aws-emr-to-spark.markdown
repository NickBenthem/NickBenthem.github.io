---
layout: post
title: "Connect AWS EMR to Redshift"
date: "2020-04-20 21:52:14 -0700"
tags: AWS EMR Redshift Spark SQL Python S3 load data from redshift databricks
author: Nick
share: true
---

# How to connect your Spark Cluster to Redshift

I'm making this post since this [Databricks redshift Github page](https://github.com/databricks/spark-redshift) seems to be abandonded by Databricks. It's pretty good - so if you need details, that's a great place to start.

To connect EMR to Redshift, you need drivers for Spark to connect to Redshift. Download the following four library JARs:

```
https://repo1.maven.org/maven2/com/databricks/spark-redshift_2.10/2.0.1/spark-redshift_2.10-2.0.1.jar
https://repo1.maven.org/maven2/com/eclipsesource/minimal-json/minimal-json/0.9.5/minimal-json-0.9.5.jar
https://repo1.maven.org/maven2/com/databricks/spark-avro_2.11/3.0.0/spark-avro_2.11-3.0.0.jar
https://s3.amazonaws.com/redshift-downloads/drivers/jdbc/1.2.41.1065/RedshiftJDBC4-no-awssdk-1.2.41.1065.jar
```

Next - you need to get these on your EMR instance. Upload the previous files into your S3 bucket (or place of choice) and lace the following code in the first cell of your EMR Juypter notebook. I stored them in my "myBucket" bucket

```
#%%configure -f
{ "conf": {"spark.jars":"s3://myBucket/spark-redshift_2.10-2.0.1.jar,"
                        "s3://myBucket/minimal-json-0.9.5.jar,"
                        "s3://myBucket/spark-avro_2.11-3.0.0.jar,"
                        "s3://myBucket/RedshiftJDBC4-no-awssdk-1.2.41.1065.jar"} }
```

This is a magic hint for the IPython notebook and will install your libraries when you first run the notebook.


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

EDIT: A reader hit trouble with uploading and used the following to troubleshoot the ivy logs. See https://stackoverflow.com/questions/66387589/aws-emr-driver-jars for more.

At this point - I wouldn't really recommend EMR - We jumped over to databricks and the results have been much better overall. There's a small increase in price - but the features are well worth the trouble.
