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
  There's two ways to think about compression:

  1. f
  1. f

For 2, let's focus on a singular image that you have a traditional warehouse structure:



![Some data we found](images/2020/01/some-data-we-found.png)
id  |  val
--|--
  |
  |
  |


![](/assets/img/2020-01-20T00-34-45.png)
