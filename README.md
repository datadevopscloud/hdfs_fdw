Hadoop (HDFS) Foreign Data Wrapper for PostgreSQL
-------------------------------------------------
This PostgreSQL extension implements a Foreign Data Wrapper (FDW) for 
[Hadoop][1] (HDFS).

Please note that this version of hdfs_fdw works with PostgreSQL and EDB 
Postgres Advanced Server 9.3, 9.4 and 9.5. Work is underway for certification 
with 9.6Beta.

What Is Apache [Hadoop][1]?
--------------
*The Apache™ Hadoop® project develops open-source software for reliable, 
scalable, distributed computing.The Apache Hadoop software library is a framework that allows for the distributed processing of large data sets across clusters of computers using simple programming models. It is designed to scale up from single servers to thousands of machines, each offering local computation and storage. Rather than rely on hardware to deliver high-availability, the library itself is designed to detect and handle failures at the application layer, so delivering a highly-available service on top of a cluster of computers, each of which may be prone to failures. The detail information can be found [here][1]. Hadoop 
can be downloaded from this [location][2]*.
  
  
What Is Apache [Hive][3]?
--------------
*The Apache Hive ™ data warehouse software facilitates querying and managing 
large datasets residing in distributed storage. Hive provides a mechanism to 
project structure onto this data and query the data using a SQL-like language 
called HiveQL. At the same time this language also allows traditional 
map/reduce programmers to plug in their custom mappers and reducers when it is
inconvenient or inefficient to express this logic in HiveQL*. 

There are two version of Hive HiveServer1 and HiveServer2 which can be downloded from this [site][4].

  
What Is Apache [Spark][11]?
--------------
The Apache Spark ™ is a general purpose distributed computing framework which supports a wide variety of uses cases. It provides real time stream as well as batch processing with speed, ease of use and sophisticated analytics. Spark does not provide storage layer, it relies on third party storage providers like Hadoop, HBASE, Cassandra, S3 etc. Spark integrates seamlessly with Hadoop and can process existing data. Spark SQL is 100% compatible with HiveQL and can be used as a replacement of hiveserver2, using Spark Thrift Server.

  
Usage
-----
  
While creating the foreign server object for HDFS FDW the following can be specified in options:
  
    * `host`: IP Address or hostname of the Hive Thrift Server OR Spark Thrift Server. Defaults to `127.0.0.1`
    * `port`: Port number of the Hive Thrift Server OR Spark Thrift Server. Defaults to `10000`
    * `client_type`:  HiveServer1 or HiveServer2. Default is HiveServer1
    * `connect_timeout`:  Connection timeout, default value is 300 seconds.
    * `query_timeout`:  Query timeout, default value is 300 seconds
  
HDFS can be used through either Hive or Spark. In this case both Hive and Spark store metadata in the configured metastore. In the metastore databases and tables can be created using HiveQL. While creating foreign table object for the foreign server the following can be specified in options:
  
    * `dbname`: Name of the metastore database to query. This is a mandatory option.
    * `table_name`: Name of the metastore table, default is the same as foreign table name.

  
Using HDFS FDW with Apache Hive on top of Hadoop
-----
    
Step 1: Download [weblogs_parse][8] and follow instructions from this [site][9].
  
Step 2: Upload weblog_parse.txt file using these commands.
  
  ```sh
  hadoop fs -mkdir /weblogs
  hadoop fs -mkdir /weblogs/parse
  hadoop fs -put weblogs_parse.txt /weblogs/parse/part-00000
  hadoop fs -cp /weblogs/parse/part-00000 /user/hive/warehouse/weblogs/
  ```
  
Step 3: Start HiveServer
  
  ```sh
  bin/hive --service hiveserver -v
  ```
Step 4: Start Hive client to connect to HiveServer
  ```sh
  hive -h 127.0.0.1
  ```
  
Step 5: Create Table in Hive
  ```sql
  CREATE TABLE weblogs (
      client_ip           STRING,
      full_request_date   STRING,
      day                 STRING,
      month               STRING,
      month_num           INT,
      year                STRING,
      hour                STRING,
      minute              STRING,
      second              STRING,
      timezone            STRING,
      http_verb           STRING,
      uri                 STRING,
      http_status_code    STRING,
      bytes_returned      STRING,
      referrer            STRING,
      user_agent          STRING)
  row format delimited
  fields terminated by '\t';
  ```
  
  Now we are ready to use the the weblog table in PostgreSQL, we need to follow 
  these steps.
  
  ```sql
  -- load extension first time after install
  CREATE EXTENSION hdfs_fdw;
  
  -- create server object
  CREATE SERVER hdfs_server
           FOREIGN DATA WRAPPER hdfs_fdw
           OPTIONS (host '127.0.0.1');
  
  -- create user mapping
  CREATE USER MAPPING FOR postgres
      SERVER hdfs_server;
  
  -- create foreign table
  CREATE FOREIGN TABLE weblogs
  (
   client_ip                TEXT,
   full_request_date        TEXT,
   day                      TEXT,
   Month                    TEXT,
   month_num                INTEGER,
   year                     TEXT,
   hour                     TEXT,
   minute                   TEXT,
   second                   TEXT,
   timezone                 TEXT,
   http_verb                TEXT,
   uri                      TEXT,
   http_status_code         TEXT,
   bytes_returned           TEXT,
   referrer                 TEXT,
   user_agent               TEXT
  )
  SERVER hdfs_server
           OPTIONS (dbname 'db', table_name 'weblogs');
  
  
  -- select from table
  postgres=# SELECT DISTINCT client_ip IP, count(*)
             FROM weblogs GROUP BY IP HAVING count(*) > 5000;
         ip        | count
  -----------------+-------
   683.615.622.618 | 13505
   14.323.74.653   | 16194
   13.53.52.13     |  5494
   361.631.17.30   | 64979
   363.652.18.65   | 10561
   325.87.75.36    |  6498
   322.6.648.325   | 13242
   325.87.75.336   |  6500
  (8 rows)
  
  
  CREATE TABLE premium_ip
  (
        client_ip TEXT, category TEXT
  );
  
  INSERT INTO premium_ip VALUES ('683.615.622.618','Category A');
  INSERT INTO premium_ip VALUES ('14.323.74.653','Category A');
  INSERT INTO premium_ip VALUES ('13.53.52.13','Category A');
  INSERT INTO premium_ip VALUES ('361.631.17.30','Category A');
  INSERT INTO premium_ip VALUES ('361.631.17.30','Category A');
  INSERT INTO premium_ip VALUES ('325.87.75.336','Category B');
  
  postgres=# SELECT hd.client_ip IP, pr.category, count(hd.client_ip)
                             FROM weblogs hd, premium_ip pr
                             WHERE hd.client_ip = pr.client_ip
                             AND hd.year = '2011'                                 
                 
                             GROUP BY hd.client_ip,pr.category;
                             
         ip        |  category  | count
  -----------------+------------+-------
   14.323.74.653   | Category A |  9459
   361.631.17.30   | Category A | 76386
   683.615.622.618 | Category A | 11463
   13.53.52.13     | Category A |  3288
   325.87.75.336   | Category B |  3816
  (5 rows)
  
  
  postgres=# EXPLAIN VERBOSE SELECT hd.client_ip IP, pr.category, 
  count(hd.client_ip)
                             FROM weblogs hd, premium_ip pr
                             WHERE hd.client_ip = pr.client_ip
                             AND hd.year = '2011'                                 
                 
                             GROUP BY hd.client_ip,pr.category;
                                            QUERY PLAN                            
                
  --------------------------------------------------------------------------------
  --------------
   HashAggregate  (cost=221.40..264.40 rows=4300 width=64)
     Output: hd.client_ip, pr.category, count(hd.client_ip)
     Group Key: hd.client_ip, pr.category
     ->  Merge Join  (cost=120.35..189.15 rows=4300 width=64)
           Output: hd.client_ip, pr.category
           Merge Cond: (pr.client_ip = hd.client_ip)
           ->  Sort  (cost=60.52..62.67 rows=860 width=64)
                 Output: pr.category, pr.client_ip
                 Sort Key: pr.client_ip
                 ->  Seq Scan on public.premium_ip pr  (cost=0.00..18.60 rows=860 
  width=64)
                       Output: pr.category, pr.client_ip
           ->  Sort  (cost=59.83..62.33 rows=1000 width=32)
                 Output: hd.client_ip
                 Sort Key: hd.client_ip
                 ->  Foreign Scan on public.weblogs hd  (cost=100.00..10.00 
  rows=1000 width=32)
                       Output: hd.client_ip
                       Remote SQL: SELECT client_ip FROM weblogs WHERE ((year = 
  '2011'))
  ```

Using HDFS FDW with Apache Spark on top of Hadoop
-----



How to build
-----
  
  See the file INSTALL for instructions on how to build and install
  the extension and it's dependencies. 
  
  
    
  
TODO
----
 1. Hadoop Installation Instructions
 2. Write-able support
 3. Flum support
  
Contributing
------------
If you experince any bug create new [issue][6] and if you have fix for that 
create a pull request. Before submitting a bugfix or new feature, please read 
the [contributing guidlines][7].
  
Support
-------
This project will be modified to maintain compatibility with new PostgreSQL 
and EDB Postgres Advanced Server releases.
  
If you require commercial support, please contact the EnterpriseDB sales 
team, or check whether your existing PostgreSQL support provider can also 
support hdfs_fdw.
  
  
Copyright Information
---------------------
Copyright (c) 2011 - 2016, EnterpriseDB Corporation
  
Permission to use, copy, modify, and distribute this software and its
documentation for any purpose, without fee, and without a written agreement is
hereby granted, provided that the above copyright notice and this paragraph 
and the following two paragraphs appear in all copies.
  
  IN NO EVENT SHALL ENTERPRISEDB CORPORATION BE LIABLE TO ANY PARTY FOR
  DIRECT, INDIRECT, SPECIAL, INCIDENTAL, OR CONSEQUENTIAL DAMAGES, INCLUDING 
  LOST PROFITS, ARISING OUT OF THE USE OF THIS SOFTWARE AND ITS DOCUMENTATION, 
  EVEN IF ENTERPRISEDB CORPORATION HAS BEEN ADVISED OF THE POSSIBILITY OF SUCH 
  DAMAGE.
  
  ENTERPRISEDB CORPORATION SPECIFICALLY DISCLAIMS ANY WARRANTIES, INCLUDING,
  BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR 
  A PARTICULAR PURPOSE. THE SOFTWARE PROVIDED HEREUNDER IS ON AN "AS IS" BASIS, 
  AND ENTERPRISEDB CORPORATION HAS NO OBLIGATIONS TO PROVIDE MAINTENANCE, 
  SUPPORT, UPDATES, ENHANCEMENTS, OR MODIFICATIONS.
   
See the [`LICENSE`][10] file for full details.
  
[1]: http://www.apache.org/
[2]: http://hadoop.apache.org/releases.html
[3]: https://hive.apache.org/
[4]: https://hive.apache.org/downloads.html
[5]: http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/SingleCluster.html
[6]: https://github.com/EnterpriseDB/hdfs_fdw/issues/new
[7]: CONTRIBUTING.md
[8]: http://wiki.pentaho.com/download/attachments/23531451/weblogs_parse.zip?version=1&modificationDate=1327096242000
[9]: http://wiki.pentaho.com/display/BAD/Transforming+Data+within+Hive
[10]: LICENSE
[11]: http://spark.apache.org/


  