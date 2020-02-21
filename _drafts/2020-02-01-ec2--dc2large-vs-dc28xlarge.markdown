---
layout: post
title: "EC2 - DC2.large vs DC2.8xlarge"
date: "2020-02-01 16:49:01 -0800"
---

# Background

At Everlane, we use AWS Redshift to handle our data warehousing - we gather information from accounting, finance, inventory states, and a bunch of other systems to allow us to make better informed decisions. We surface this information mainly in Looker, but allow adhoc connections to our database.  We'd been running into issues with performance on our main redshift cluster - the leader node was maxing out at 100% CPU, especially on mondays.
{% figure caption: "hello world"  %}
![Back then](/assets/img/ec2--dc2large-vs-dc28xlarge/back-then.png)|
{%endfigure%}


Our redshift cluster absorbs data from a variety of connection types. We land data into our redshift cluster using from S3 or directly from SQL instances. Our system constantly syncs data from these imports. To perform our transformation, we perform SQL transformations or load data into EC2 instances and perform compute outside of redshift and then load the data back into redshift using INSERT/COPY operations for analytics purposes. For our reporting platform we use a combination of Looker, Mode, and adhoc SQL tools to analyze the datasets.

At a high level, here's a map of how we consume data into redshift an
# Leader nodes and you
One of the key features of redshift is it's ability to be distributed across multiple nodes. There has to be some way to dispurse your SQL queries across the number of nodes. We want to have a single endpoint available for our SQL clients to talk to, and that's where the leader node comes in.


The leader node is, in general, responsible for the following activities:

1. Handling database connections to redshift
2. Orchestrating redistribution of data between nodes
3. Caching results
4. [Leader node only functions](https://docs.aws.amazon.com/redshift/latest/dg/c_SQL_functions_leader_node_only.html)

When you spin up a multi-node redshift cluster (i.e., more than 1 node), redshift gives you a leader node at no additional cost. As of this article, that leader node is the same size as the compute nodes - which means by upgrading your cluster node type, you upgrade your leader node. With
dc2.large nodes, you get a leader node with
2vCPU	15 GiB	0.16TB SSD	0.6 GB/sec
DC2.8xlarge nodes, you get
32vCPU	244 GiB	2.56TB SSD 7.5 GB/sec
versus
#

These were the result of taking queries from our production database and replaying them against a freshly spun up copy of redshift.
![48 dc2.large Nodes - query](/assets/img/ec2--dc2large-vs-dc28xlarge/48-dc2-nodes.png)


![3 dc2.8xlarge nodes - query](/assets/img/ec2--dc2large-vs-dc28xlarge/3-dc2-8xlarge-nodes.png)


We also tested a singular ETL process (mixture of read / write)
![48 dc2.large nodes ETL ](/assets/img/ec2--dc2large-vs-dc28xlarge/48-dc2-large-nodes-etl.png)


 ![3 dc2.8xlarge nodes ETL](/assets/img/ec2--dc2large-vs-dc28xlarge/3-dc2-8xlarge-nodes-etl.png)

The 48 dc2.large nodes took 11 minutes, which the 3 dc2.large nodes took 9 minutes to complete.

This is a graph of our average query time around the time of our cutover - we cut over on the 18th and have seen significant improvement in our query times.
![Swapping over](/assets/img/ec2--dc2large-vs-dc28xlarge/swapping-over.png)

![Our CPU now](/assets/img/ec2--dc2large-vs-dc28xlarge/our-cpu-now.png)

![Back then](/assets/img/ec2--dc2large-vs-dc28xlarge/back-then.png)

A huge thank you to Gisela Morrone who helped perform a lot of testing and analysis.
--

https://www.intermix.io/blog/amazon-redshift-architecture/
