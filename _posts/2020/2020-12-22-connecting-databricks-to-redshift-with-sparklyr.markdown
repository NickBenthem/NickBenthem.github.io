---
layout: "post"
title: "Connecting Databricks to Redshift with SparklyR"
date: "2020-12-22 09:05"
---

Databricks gives documentation hooking up Spark with Redshift using the raw Spark libraries, but not with SparklyR, which gives some great functions you want (notably - dplyr syntax).

Unsurprisingly, you need to use the `sparklyr::spark_read_jdbc` command, but critically, you need redshift jdbc42 driver installed - you can install with Maven or follow the instructions at AWS (https://docs.aws.amazon.com/redshift/latest/mgmt/configure-jdbc-connection.html#download-jdbc-driver). Once it's installed - you can just use the following function to get your data - the critical component here is the `driver = "com.amazon.redshift.jdbc42.Driver"` piece. Your jdbcUrl needs to be of the form `jdbc:postgresql://endpoint:port/database`

i.e.,

```
jdbc_url <- jdbc:redshift://examplecluster.abc123xyz789.us-west-2.redshift.amazonaws.com:5439/dev
```

Then you can create a function to load in your data as:
```
read_from_redshift_spark <- function(sql_string, sc, table_name, jdbcUrl, redshift_password) {
  df <- sparklyr::spark_read_jdbc(sc,
                    name = table_name ,
                    options = list(url = jdbcUrl,
                                   user = redshift_user,
                                   driver = "com.amazon.redshift.jdbc42.Driver",
                                   password = redshift_password,
                                   dbtable = sprintf("(%s)", sql_string) ))
  return(df)
}
```

Wrapping it all together -

```
library(SparklyR)
library(dplyr) # order matters - https://www.ucl.ac.uk/~uctqiax/PUBLG100/2015/faq/masking.html
config <- spark_config()
sc <- sparklyr::spark_connect(method = "databricks", config = config) # Connect locally
read_from_redshift_spark <- function(sql_string, sc, table_name, jdbcUrl, redshift_password,redshift_user) {
  df <- sparklyr::spark_read_jdbc(sc,
                    name = table_name ,
                    options = list(url = jdbcUrl,
                                   user = redshift_user,
                                   driver = "com.amazon.redshift.jdbc42.Driver",
                                   password = redshift_password,
                                   dbtable = sprintf("(%s)", sql_string) ))
  return(df)
}

jdbc_url <- jdbc:redshift://examplecluster.abc123xyz789.us-west-2.redshift.amazonaws.com:5439/dev
redshift_password <- "passwordhere"
redshift_user <- "userhere"
dataset_test <- read_from_redshift_spark("select * from my_table",sc,"my_table_name_spark_sql",jdbcUrl,redshift_password,redshift_user)

dataset_test %>% select("column1")
```


And bam - you have dplyr syntax in Spark.
