---
layout: post
title: "Killing long running redshift connections"
date: "2020-02-03 12:10:38 -0800"
---

We recently ran into an issue where we needed to kill database connections to redshift. To do so, we just wrote the following script.


We recently ran into an issue where

```
CREATE OR REPLACE PROCEDURE "admin"."krombopulos_michael" (kill_longer_than_x_hours integer)
AS $$
--Oh boy...
DECLARE
reports RECORD;
BEGIN
        FOR reports IN select * from stv_sessions where starttime < DATEADD(HOUR,-1*kill_longer_than_x_hours,GETDATE()) AND user_name <> 'rdsdb' /*system name*/
        LOOP
        EXECUTE $_$ SELECT pg_terminate_backend( $_$||  reports.process||  $_$ ) $_$;
        END LOOP;
END;
$$
 LANGUAGE plpgsql
 ```

 ![Michael Krombopulus](/assets/img/killing-long-running-redshift-connections/michael-krombopulus.gif)
