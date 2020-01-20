---
layout: post
title: "Redshift Compression"
date: "2020-01-17 14:18:00 -0800"
---

One of the things we've been dealing with at Everlane has been dealing with some large scale tables. Redshift and Postgres have some pretty cool technology to allow you to compress your tables and get both faster query run time as well as reduce the space you consume.

# A brief primer on compression


![](images/2020_01_17_redshift_compression/2020-01-20T00-34-45.png){:height="36px" width="36px"}
