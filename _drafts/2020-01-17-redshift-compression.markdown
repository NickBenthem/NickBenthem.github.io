---
layout: post
title: "Redshift Compression"
date: "2020-01-17 14:18:00 -0800"
---

One of the things we've been dealing with at Everlane has been dealing with some large scale tables. Redshift and Postgres have some pretty cool technology to allow you to compress your tables and get both faster query run time as well as reduce the space you consume.

# A brief primer on compression

Why do we want to have compression in the first place? After all, we usually zip up files to send them to others, or think there'll be a performance hit with compressing our data (such as if you had on your local machine)

Due to how redshift stores data, compression is actually desired inside our our databases.

## How compression works

Let's look at one way how we would do this in a SQL database given the following table:

![Some example data](/assets/img/redshift-compression/some-example-data.png)

Now, many of these values are going to be repeated often. If we wanted to be efficient in our computations, we'd want to take advantage of these repititions. Each time we store a value, we currently have to store the entire value of our field in each and every row. If we assume 2 bytes per row ([A slightly simplified assumption](https://docs.aws.amazon.com/redshift/latest/dg/r_Character_types.html)), to store the team values of "Detroit Lions", we'd need `len('Detroit Lions')*2 = 13*2 = 26 bytes` per row. This seems rather innefficient though. If we have a table containing millions of values, we'll be storing a bunch of information redundantly.

Traditionally, in the history of data warehousing, we would implement a snowflake schema to speed up our values:

![dim_warehouse](/assets/img/redshift-compression/dim-warehouse.png)

Where we'd have two tables: `dim_position` and `dim_team`, which would look something like:

![The dim_warehouse tables](/assets/img/redshift-compression/the-dim-warehouse-tables.png)

This gains us a pretty huge advantage. If we use something similar to a `INT` datatype, we now are just storing `2 bytes` per value, **a 92% reduction in size** over the previous system. If our table is gigabytes in size, we'll need to read far less data, allowing our aggregation of data to be much faster as we have to process less data for each row.

However, we now have two problems:

1. we have to create and maintain these dimension tables
2. Any sort of user-readable view of these tables using SQL requires you to create views on top doing a join to take your transformed snowlflake schema and restore your previous world.

This of course is problematic. Any sort of compression we do now needs requires us to perform non-business add value to our work by creating this schema, ensuring that all values become compressed, and generally slowing us down.

## Enabling this compression using redshift

Thankfully, we're able to gain these sorts of storage benefits in redshift, all without having to generate a full data warehouse scheme and create user views.

Let's go through compressing our top tables in redshift. We'll be using AWS' compression tool.
