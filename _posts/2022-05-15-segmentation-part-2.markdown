---
title: Customer Segmentation at Glovo - Part 2
excerpt: Customer Segmentation at Glovo - System design  
last_modified_at: 2022-05-15
categories:
  - Audience targeting, microservices, big data
tags:
  - Audience targeting, microservices, big data
toc: true
toc_label: Customer Segmentation - System design 

toc_icon: code
toc_sticky: true
---

# Part 2: Segmentation system design 


In part 1 we talked about the customer segmentation needs at Glovo and why we need to customize the customer's UX, for example when targeting specific audiences for certain discounts. We also discussed the need to separate the segment's calculations task from the segment's restful API. In this part 2, we will dive into the two components, explaining their respective responsibilities and the overall architecture.

We named the two components "Core" and "Store"; the Core is responsible for performing Segment calculations in a scheduled fashion (every X minutes/hours), based on the segment definitions provided; the Store is in charge of segment configuration management and auditing, and also serving the calculated segments to the rest of the system. Also, between these components, we need integrations to exchange segment configurations and segment results, to finally expose those results to the external parties.

The full architecture looks like this:

{:refdef: style="text-align: center;"}
![system design book](/assets/images/segmentation-diagram.png)
{: refdef}

Going into details for each component, integrations, and how the data flows:

## Core

For the data calculations, we decided to leverage the existing Glovo's [Data Lake](https://en.wikipedia.org/wiki/Data_lake) and [Big data](https://en.wikipedia.org/wiki/Big_data) tools that we already had. There were several reasons for this decision:

- We knew we needed to query data from multiple domains, initially orders, users, and products but possibly many others in the future as we defined new segment types. The Data Lake, built on S3, solved this requirement out of the box: it was a centralized store already gathering data from most of the domains in the system. Although data in the Data Lake was not live data (it was refreshed every half an hour), we didn't need live data for most of the segmentation cases.
- Performing the segment calculations would involve hundreds of millions of rows, and terabytes of data. So it was clear to us that the best way to perform these calculations was using Big Data processing tools. At Glovo we use [AWS EMR](https://aws.amazon.com/emr/) and [PySpark](https://spark.apache.org/docs/latest/api/python/) to run calculations on the Data Lake.

The full stack for the core included also Jenkins and Luigi. New segment calculations would be put in an [S3](https://aws.amazon.com/s3/) prefix for the given DateTime, and historical data could be eventually deleted by using [object expiration policies](https://docs.aws.amazon.com/AmazonS3/latest/userguide/lifecycle-expire-general-considerations.html).

## Store

As this component is responsible for segment configurations management and also serving segment calculations, it includes several parts: the backend exposing a restful API, an RDBMS holding the segment configurations, an Elastic search cluster to store and serve the segments calculations to the backend, and Redis for caching customer-segments queries to achieve the low latency needed.

As explained before, the Core is responsible for segment calculations, but we said that the Store would serve the calculations, since it has the appropriate design and technologies to do so, scaling with constantly low latency. In order to move the calculated segments from the Store to the Core, we designed an ingestion process.

## Store clusters: Api and Ingestion

Given the very different workloads and responsibilities of ingesting data coming from the Core versus serving that data to the outside domains, we decided to have separate clusters for each task, splitting reads and writes. This would allow us to ingest in a decoupled and efficient way, scaling each cluster independently.

The ingestion flow would work as follows: the PySpark logic in the Store puts a specific flag file in a predefined S3 prefix, indicating that a calculation has finished; an S3 to [SQS](https://aws.amazon.com/sqs/) integration would put a message in an SQS queue; the Store Ingestion cluster would find the message in the queue, read it and start reading the corresponding Segment calculations from S3, persisting them in ElasticSearch in a new index, to later perform an [Elastic alias](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-aliases.html) switch from the previous index, avoiding downtime. The Elasticsearch insertions should be efficient, using the [bulk API](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-bulk.html), parallelization, and retries in the case of errors. Also, the ingestion is a process that should happen at most once in a given timeframe, so we had to deal with the at-least-once nature of SQS queues by using a Redis lock and tweaking the message [visibility timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html) as the ingestion proceeded.  

## Elastic search

To select the proper data store for segment calculations search, we had to take into consideration multiple factors. First and foremost, is the volume of data. We expected thousands of segments to be created by the business and partners already for our phase 1, and dozens of thousands once we would add more segment types. Also, each one of these segments could have millions of customers. In other words, we were handling large data. The second critical factor was that a given customer could belong to hundreds or even thousands of segments in our system; yet different clients would be interested only in some of those segments for the customer: some would only care about segments for a given store, others about store categories (pharmacies but not restaurants), and so on. This meant we needed a search engine that could scale horizontally, and capable of querying by multiple, unbounded fields, and in future use-cases using text search. We agreed that this engine had to be [ElasticSearch](https://www.elastic.co/es/?ultron=B-Stack-Trials-EMEA-S-Exact&gambit=Stack-Core&blade=adwords-s&hulk=paid&Device=c&thor=elasticsearch&gclid=Cj0KCQjwm6KUBhC3ARIsACIwxBioYniNLnDpcTPIcOzFfCmwoZ2FVsBmK8als7BUqBmvVXQM8G7BOK0aAv0oEALw_wcB).

Using ElasticSearch we had to size the number of replicas and shards, we carefully selected the proper data types for each indexed field (such as using [keyword](https://www.elastic.co/guide/en/elasticsearch/reference/current/keyword.html#keyword-field-type) type for id fields) to make sure the corresponding [inverted index](https://www.elastic.co/es/blog/found-indexing-for-beginners-part3#inverted-index) would work as expected. Other relevant enhancements we had to do included tweaking the elastic index [refresh rate](https://www.elastic.co/guide/en/elasticsearch/reference/current/tune-for-indexing-speed.html#:~:text=By%20default%2C%20Elasticsearch%20periodically%20refreshes,in%20the%20last%2030%20seconds.), using [term queries](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-term-query.html) from the  Spring elastic client, and partitioning the large docs to remain below the [maximum document size](https://www.elastic.co/guide/en/app-search/current/limits.html).

## Redis Cache in the API Cluster

We had committed to an SLO of p99 < 50ms and to achieve it we leveraged the eventually-consistent nature of the segment calculations we were serving: we could cache heavily without worrying so much about the [time to live (TTL)](https://redis.io/commands/expire/#:~:text=Normally%20Redis%20keys%20are%20created,instance%20using%20the%20DEL%20command.) of the entries or the cache invalidation. With Redis as a simple [cache-aside](https://docs.aws.amazon.com/whitepapers/latest/database-caching-strategies-using-redis/caching-patterns.html) strategy, we cached the most frequently accessed customer segments, and not only did we drop the latency to less than 10ms, but we also reduced the load on the Elastic cluster. 

{:refdef: style="text-align: center;"}
![system design book](/assets/images/cache-aside.jpg)
{: refdef}

If you want to know the results and the current status of this project go and read part three!
