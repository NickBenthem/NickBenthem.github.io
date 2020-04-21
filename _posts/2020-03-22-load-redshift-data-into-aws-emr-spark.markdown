---
layout: post
title: "Connect AWS EMR to Redshift"
date: "2020-03-22 20:18:36 -0700"
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
%%configure -f
{ "conf": {"spark.jars":"s3://myBucket/spark-redshift_2.10-2.0.1.jar,s3://myBucket/minimal-json-0.9.5.jar,s3://myBucket/spark-avro_2.11-3.0.0.jar,RedshiftJDBC4-no-awssdk-1.2.41.1065.jar"} }
```

This is a magic hint for the IPython notebook and will install your libraries at the beginning of the notebook.
