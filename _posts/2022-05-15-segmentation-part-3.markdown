---
title: Customer Segmentation at Glovo - Part 3
excerpt: Customer Segmentation at Glovo - Rollout, results, and the future of segmentation

last_modified_at: 2022-05-15
categories:
  - Audience targeting, microservices, big data
tags:
  - Audience targeting, microservices, big data
toc: true
toc_label: Customer Segmentation - Rollout, results, and the future of segmentation

toc_icon: code
toc_sticky: true
---

# Part 3: Rollout, results, and the future of segmentation

Before we rolled out the project, we had a large checklist to validate. The segmentation project encompasses multiple components and is used by multiple clients, so we needed an exhaustive rollout plan to minimize the chances of failure, and to be ready for possible rollbacks. We also required proper documentation in a centralized place, including the design, observability links, and [OpenAPI](https://swagger.io/specification/) Specification of our Restful endpoints. and all the relevant resources to allow the team members and new joiners to learn and contribute to the project.

### Launch Plan

We assembled a launch plan covering business and technical acceptance criteria, rollout and rollback step-by-step guides, and dashboards to monitor the system metrics during the first days after the release. Analysts performed cross-validations with users that met segment criteria. We also agreed on a gradual rollout starting with a single city, then expanding to the whole country, and finally to other countries, so that the gradual increase in the traffic could unveil any errors or performance issues early. AB tests were used to validate the results. 


### Performance testing

To accommodate the load from the client domains, we had to define initial sizes for our API, Ingestion, ElasticSearch, and ElastiCache clusters. Since we didn't have a proper load testing tool by that time we used JMeter to perform a first load test on the service, and later we enabled shadow traffic from the real clients, with all the proper observability around the key metrics. We adjusted the clusters node minimum and maximum node count, the node instance sizes, and storage types, among other things, until we reached the performance we required. Later on, autoscaling would adapt according to the load.

### Resiliency

We couldn't risk harming our existing clients' SLOs, and we had to define our Segmentation SLO respecting the downstream services. On the ingestion process side, resilience was needed on reading segment parts from S3, and also when persisting to ElasticSearch; moreover,  we didn't want to halt the entire ingestion of terabytes of data when a single part out of thousands would fail, so we set up error tolerance rates. On the Elastic side, we wanted to keep the previously ingested data from old indexes for a while, in case the new index wouldn't be created for failures during segments calculations or ingestion: to achieve this, we used [indexes lifecycle policies](https://www.elastic.co/guide/en/elasticsearch/reference/current/getting-started-index-lifecycle-management.html#ilm-gs-create-policy). Finally, the client's synchronous calls to the API cluster were protected with [circuit breakers](https://martinfowler.com/bliki/CircuitBreaker.html), timeouts, and retries.

### Observability

To deliver the project proper observability was critical, including logs, metrics, dashboards, monitors, and alerts. Initially, the alerts would point to a dedicated [Slack](https://slack.com/intl/es-es/) channel we created, to be promoted later to [Pagerduty](https://www.pagerduty.com/) to reach the on-call engineer. On-call [run-books](https://www.pagerduty.com/resources/learn/what-is-a-runbook/) were also updated.

Main observability resources:

- Datadog's [Application Performance Monitoring (APM)](https://www.datadoghq.com/dg/apm/ts-benefits-os/?utm_source=advertisement&utm_medium=search&utm_campaign=dg-google-apm-emea-general&utm_keyword=application%20performance%20monitoring&utm_matchtype=p&utm_campaignid=15419168583&utm_adgroupid=131788862153&gclid=CjwKCAjw4ayUBhA4EiwATWyBrj374lFcZmYeTkTWO_ptkxUE3Xat4CXbiSWcNPFcBpQBTdiv-6fDfRoCOEgQAvD_BwE) on our different clusters, including API, Ingestion, and ElasticSearch.
- General segmentation metrics dashboard, including endpoints, ingestions, segments calculations, SQS queues, and HTTP Client's health
- [Kibana](https://www.elastic.co/es/kibana/) to keep track of the index's status, including their size, document count, and policies, and to have a handy way to execute queries against Elastic.


### Real-time data

What about live data? We knew that in the future, some use cases could require data to be live. To accomplish this we had to consume live events from our [Kinesis streams](https://aws.amazon.com/kinesis/data-streams/) to update our segment calculations in real-time. Although doable, this was left out of scope and postponed for a future iteration of the project, in favor of limiting the scope to deliver the core segment types that would bring the most value to the business.

### Rollout results

The production rollout was a success, no downtime nor rollbacks were needed; the application was already serving up to 8.56 requests per second, with a p99 of 12.05ms on average, and p75 avg 1.23ms.

![system design book](/assets/images/datadog-dashboard.jpeg)
{: refdef}

### Alternatives and why they were discarded

During the project, the design of the components and the decisions about technologies we had multiple options on the table; some of them didn't fit well and were quickly discarded, while others were compelling but had tradeoffs.

For serving the data, we had initially considered [DynamoDB](https://aws.amazon.com/dynamodb/) as a scalable, low-latency store to serve customer segment data. We discarded this Database because of the rigid querying capabilities, and also the cost of constantly overriding the data on each ingestion.

At the time of ingesting data into ElasticSearch, we had big candidates like [Logstash](https://www.elastic.co/es/logstash/) and [AWS DMS](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Target.html). We didn't go for these options because we wanted to keep full control of the ingestion process, with full customization and schema transformation capabilities.

The segmentation project is currently being adapted to support new types of segments for other parts of the Glovo system. There is also an ongoing project to create a generic set of tools on top of the data lake that will simplify the queries and calculations.
