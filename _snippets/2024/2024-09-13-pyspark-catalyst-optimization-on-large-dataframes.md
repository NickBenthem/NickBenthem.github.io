---
layout: snippet
title: PySpark Catalyst Optimization on large DataFrames
date: 2024-09-13 16:25 -0400
---

Running some operations on DataFrames with a large number of columns, or a large set of data, can get slow. PySpark does lazy evaluation, which means the computationally expensive things are delayed until the very end when needed (often when writing the data, `.collect()`, or by `.count()`). 

However, PySpark's optimization engine, Catalyst, will need to do some calculations before then when performing operations on the DataFrame itself. Things like adding/removing a column, or iterating over properties of the DataFrame require PySpark to do work to compute the state of the DataFrame up to that point.

Take for instance when performing a rename operation:

```
for old_col in cols:
    new_col = f"new_{old_col}"
    df = df.withColumnRenamed(old_col, new_col)
```

PySpark Catalyst will trigger the optimization on each `withColumnRenamed` iteration in the loop. If the number of columns / size of data is large, **this can get very, very slow**. 

It's better to batch the operations when possible. If you're on a recent version of Spark (3.4.0+), you can utilize [withColumnsRenamed](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.DataFrame.withColumnsRenamed.html), which allows batching. Using this batch functionality, the above can be rewritten as 

```
# Performed entirely in Python, without PySpark calls.
renamed_columns = {col : f"new_{col}" for col in cols}

# Perform a singular PySpark call.
df = df.withColumnsRenamed(renamed_columns)
```

Here's a quick demonstration of how bad this can get:

- With 150 columns and 100 rows: 
    - The for loop takes 4.6962 seconds:
    - The batch approach takes 0.3613 seconds
- With 250 columns and 100,000 rows:
    - The for loop takes 9.4649 seconds:
    - The batch approach takes 0.8916 seconds

`withColumnsRenamed` is one example, but this concept also applies to when you iterate over the columns in a PySpark DataFrame multiple times. If you know that you can re-use a cached list of the PySpark columns, you should be able to save some time.

Demonstration code below:
```
from pyspark.sql import SparkSession
from pyspark.sql import functions as F
import time

# Create a Spark session
spark = SparkSession.builder.appName("Demo").getOrCreate()

# Generate a DataFrame with 1000 columns
cols = [f"col_{i}" for i in range(150)]
data = [tuple(i for i in range(150)) for _ in range(100)]  # 10 rows of data


# Inefficient approach: using a for loop with .withColumnRenamed

df = spark.createDataFrame(data, schema=cols)
start_time = time.time()  # Start the timer

for old_col in cols:
    new_col = f"new_{old_col}"
    df = df.withColumnRenamed(old_col, new_col)

df.count() # Force materialization.

end_time = time.time()  # End the timer
print(f"Inefficient approach took {end_time - start_time:.4f} seconds")

# More efficient approach: using a batch operation.
df = spark.createDataFrame(data, schema=cols)
start_time = time.time()  # Start the timer

# Performed entirely in Python, without PySpark calls.
renamed_columns = {col : f"new_{col}" for col in cols}
# Perform a singular PySpark call.
df = df.withColumnsRenamed(renamed_columns)

df.count() # Force materialization.

end_time = time.time()  # End the timer
print(f"Efficient approach took {end_time - start_time:.4f} seconds")
```

