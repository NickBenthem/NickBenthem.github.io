---
layout: post
title: "Data Quality at the forefront"
date: "2020-01-30 09:32:38 -0800"
---

There are quite a few systems that Everlane uses to coordinate our business - from Accounting systems, to our E-commerce website, to various inventory logistics tools. We need to consume and transform the data between multiple systems, multiple data types, all while adding new information and

We've all heard of "Garbage in, Garbage out", but most ETL systems are designed to only check that something runs - not that the result are within your bounds.

I'm going to seperate the discussion into two categories - one is (almost) entirely backend, the other is based around data teams.

# Data Quality

# Data Integrity

Data Integrity is

Infact, it helps to think of data like water flowing through your system at this point. Data integrity is the ability for your flow intact through the pipes (did we lose any rows? Are the data types correct? Did we end up getting the data in a timely manner). You're more than likely that you're doing data integrity at this minute by monitoring your airflow jobs, using proper sanitation checks on the data types for columns in your SQL warehouse.


Data Quality is similar to water quality - would you want to consume this data, or did it get mucked up along the way. For instance, you can detect whether upstream systems are giving you data you wouldn't expect: for instance, if you increased your sales by 300% in a single day, you can have all systems report that it processed successfully, but you as a human know that shouldn't be the case.


# how we structure our tests

Currently, we use a variety of methods to test. One method we're particularly excited about is Great Expectations. SELECT
    po.number                                                   AS po_number,
    var.sku                                                     AS sku,
    var.upc                                                     AS upc,
    mc.price                                                    AS price,
    poi.expected_quantity                                       AS qty,
    CASE WHEN poi.is_launch = 'False' THEN prod.display_name END     AS style_description, -- need a row_number function for this
    pf.name                                                              AS factory_name,
    CASE WHEN poi.is_launch = 'False' THEN ps.short_name  END       AS short_name,
    CASE WHEN poi.is_launch = 'False' THEN prod.name   END         AS color_name        -- need a row_number function for this

FROM everlane_everlanedev.purchase_order_items


This is where

Culture around fixing the issue:
Knowing that your data is wrong isn't the end of the world
