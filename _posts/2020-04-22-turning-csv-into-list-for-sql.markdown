---
layout: post
title: "Turning csv into list for SQL"
date: "2020-04-22 11:53:10 -0700"
---

So this is something I've been doing for a long time - if you have a CSV of values and don't want to write a Python script to read in the results and execute SQL, you can go ahead and use Excel/GoogleSheets/{your favorite spreadsheet program} to create this into a comma seperated list.

Open your data in Excel and place your data in the first column. Add a second column with

```
=CONCAT("'",A2,"',")
```

Then expand downwards

![ExcelExample](/assets/img/turning-csv-into-list-for-sql/excelexample.png)

Bam. You've now got a list you can use in SQL.
