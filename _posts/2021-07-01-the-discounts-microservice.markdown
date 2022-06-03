---
title: The Discounts microservice
excerpt: Microservice design, monolith to microservice migration, distributed systems
last_modified_at: 2021-07-01
categories:
  - Distributed systems
tags:
  - Distributed systems
toc: true
toc_label: The Discounts microservice
toc_icon: code
toc_sticky: true
---

Introduction to [Glovo](https://about.glovoapp.com/):
> Glovo is a Spanish tech company that provides innovative solutions to connect customers, businesses and couriers to provide delivery services, and it is the fastest-growing multicategory player in Europe, Western Asia and Africa.



# Discounts inception

This post is a replica of my [original post](https://medium.com/glovo-engineering/the-discounts-microservice-14f4d4e4d6c9) in Glovo's Medium blog.

The Discount microservice was an interesting and challenging project. I was appointed to work on the architecture design and the analysis of the extraction process from the monolith. Later after the analysis, we would work on the implementation of an MVP.

Glovo had started the monolith to microservices path some months ago, and there was no standardised path nor much internal experience on the topic; Discounts would be the first microservice in our cluster. On the other hand, business stakeholders were not ready to halt feature development for a microservice extraction.


# Discounts at Glovo

Our team had analysed and defined the discounts domain boundaries and responsibilities, and now the goal was to create this domain. At Glovo, discounts come from diverse origins, and we needed a centralised place to govern them all, supporting the same rulesets to enable existing and future discount types efficiently. An example would be enabling discounts for a given set of stores, discounts for only a subset of store types (like pharmacies but not supermarkets), or discounts for prime users only. So, for the Discounts inception, it was paramount to provide an MVP with an extensible architecture capable of covering upcoming features in our roadmap.

## Why do we need Microservices?

The long-term goal of the whole tech team was to split the monolith into smaller cohesive units, around well-defined [bounded contexts](https://martinfowler.com/bliki/BoundedContext.html), because the big ball of mud had become an increasing burden, impacting development experience as well as business, causing downtime as it kept on growing in complexity and data.

- [SPoF](https://en.wikipedia.org/wiki/Single_point_of_failure): a monolithic architecture where multiple domains coexist in the same deployment unit and also share the same underlying infrastructure tends to be more fragile than a distributed architecture using resiliency patterns
- Flaky tests halt the CI pipeline every day.
- Heavy local development environment.
- The monolithic relational database (Aurora MySQL) with ever-worsening issues (long-running transactions, deadlocks, high commit latency, among others) and reaching the maximum vertical scaling limit.
- Overall, slow [Software Development Life Cycle](https://cio-wiki.org/wiki/Software_Development_Life_Cycle_(SDLC)).

Also moving away from the monolith would allow our team to be faster, more flexible, resilient, and scale independently.

Where we were:

{:refdef: style="text-align: center;"}
![system design book](/assets/images/where-we-were.jpeg)
{: refdef}

Long term goal (simplified):

{:refdef: style="text-align: center;"}
![system design book](/assets/images/where-we-wanted-to-be.jpeg)
{: refdef}

## What about a modular monolith?

At Glovo we have a [modular monolith](https://modularmonolith.net/), and for this project, we had considered the alternative of modularising our domain as a first step before moving on with the extraction to a microservice. The main advantage of starting with a module was the lower engineering effort, and shorter time for a milestone; yet the disadvantages of remaining in the monolith outweighed the advantages:

Our new discounts DB model and data would put a strain on the Monolithic DB, and any discounts incident arising inside the monolith could bring the whole system down. That’s why we decided to go straight for a microservice.


## Code reuse in a distributed architecture


Most of the code in our new domain was brand new, but not all of it: we had specific logic we needed to migrate. This brought the next question: how should we manage code reuse within our distributed architecture? We knew several strategies, like shared libraries, shared services, service mesh sidecars, and code replication. To choose one, we had to contrast them against our case: we had a discount logic inside a few classes, and we knew that this code wouldn’t be changed during our extraction phase. It looked like a very good match for the code replication strategy!

In the code replication strategy, shared code is copied into each service source code (in our case the discounts service and the monolith). This strategy was certainly going against the [DRY principle](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself). We had to be cautious and confident that changes or bug fixes wouldn’t come, as it would have been very risky to update the replicated code in different places. But for our case, we were confident there was not much risk; plus any other alternative, such as creating a shared library would have been overkill. And so we decided to copy these classes from the monolithic codebase to the microservice. After the MVP was delivered successfully, we were safe to remove the duplicated code from the monolith.



## Key analysis and design decisions

### Database Analysis and Choice

Much of the traffic coming from customers would go through our new service, most of it for store navigation and checkout. The service would be read-heavy.

We estimated approximately 1.63M rows for the initial phase of the roadmap, and with the expected growth in the next 3 years would grow to 25.3M. Since we would extract each discount type from the monolith one by one, our DB model wasn’t defined and would evolve. These factors made us go for a relational Aurora MySQL DB. The use of Indexes would be crucial, and denormalization was among our options as well. To help with the load and latency we proposed to use [ElastiCache-Redis](https://aws.amazon.com/es/elasticache/redis/). We had a strategy for table partitioning by region as well.

### Event-driven design

The Discounts service would consume [Kinesis events](https://aws.amazon.com/kinesis/?nc1=h_ls) from each different discount provider to transform this information into its own model and persist it in the Datastore. By using remote events, new discounts would become available soon after being created in the provider’s domains, but not instantly. With this event-driven architecture, we had to embrace eventual consistency, a necessary sacrifice to allow decoupling and scaling.

### Eventual consistency challenges

As the service would consume events to stay up to date, the eventual consistency brought the challenge of inconsistent states. To handle these cases we came up with a strategy to temporarily disable discounts in use, by using [distributed locking](https://redis.io/docs/reference/patterns/distributed-locks/).

A simplified diagram of the happy path:

{:refdef: style="text-align: center;"}
![system design book](/assets/images/happy-path.jpeg)
{: refdef}

In the unhappy path, things go wrong (as they normally do) and Customer 1 doesn’t release the lock, the lock expires.

{:refdef: style="text-align: center;"}
![system design book](/assets/images/unhappy-path.jpeg)
{: refdef}

For these cases, we need [compensating transactions](https://en.wikipedia.org/wiki/Compensating_transaction) that can trigger flows to correct the situation.

### More distributed systems challenges

Moving to a distributed architecture meant leaving ACID transactions behind and embracing eventual consistency. But also we had to account for the new network hops and the issues they bring. Discounts write flows would consume remote events to transform and persist them locally as discounts, but our read flows (i.e serving data) would be synchronous: Discount clients had to retrieve discounts from our Restful API, and we had to comply with an SLA of 99.99%, as the discounts were being served to most of the flows within the Glovo app.

Another mandatory requisite was to perform a smooth transition to our service, with no downtime: to achieve that we had to make sure none of the components were misconfigured or failing, plus we needed to test the actual latency between the monolith and our service. By then we didn’t have proper performance or load testing tools at Glovo, so we decided to use a traffic mirroring technique.


### Introducing the Discounts SDK and Stability patterns

To deal with synchronous communication issues and also have a traffic mirroring tool, we created our Discounts SDK, which included an HTTP Client wrapped with [resilience4j](https://resilience4j.readme.io/docs) to use some of the [stability patterns](https://github.com/csabapalfi/release-it/blob/master/3-stability_patterns.md) it provides, in particular circuit breakers, timeouts, and fallback responses to the local discounts search. We leveraged these patterns from the moment we started sending the shadow traffic.

With the shadow traffic, we gathered business and performance metrics, validating the correctness of the discount responses by comparing against local discounts search, and checking the latency was acceptable and no unexpected errors or misconfigurations were introduced. Finally, with our SDK we could control the gradual shift between the deprecated local monolithic discounts search and the new remote discounts in a [branch by abstraction](https://martinfowler.com/bliki/BranchByAbstraction.html) fashion. To achieve this and also to have a kill switch we used [feature toggles](https://martinfowler.com/articles/feature-toggles.html). That’s how we performed production load and correctness tests, apart from the Unit and Integration tests we had in our CI pipeline.


# Other interesting tasks

Some relevant tasks that we delivered:

- Setup of the Continuous blue/green deployments using [Spinnaker](https://spinnaker.io/)
- Adding [Datadog](https://www.datadoghq.com/lpg/?utm_source=advertisement&utm_medium=search&utm_campaign=dg-google-brand-ww&utm_keyword=%2Bdatadog&utm_matchtype=b&utm_campaignid=9551169254&utm_adgroupid=95325237782&gclid=Cj0KCQjwyMiTBhDKARIsAAJ-9VszShDTIVbj0WhGZUScKt6W0IPzhu8Dn8TElbB83o7anqfqDqWgQiIaAi6rEALw_wcB) observability and monitoring for client and server errors, remote event consumers, centralised metrics dashboard, remote traffic results, and Circuit breaker errors. This all using [infrastructure as code](https://en.wikipedia.org/wiki/Infrastructure_as_code) with Terraform
- [Flyway](https://flywaydb.org/) migrations setup


# Outcome

After the MVP was successfully delivered and fully enabled in production, we measured additional technical and business improvements:

- Removed almost 170 [Queries Per Second(QPS)](https://en.wikipedia.org/wiki/Queries_per_second) from the writer instance of the monolithic DB, and 440 QPS from the readers
- Test Suite execution time went from more than 30min ( the usual monolith test times) to less than three minutes in our new CI pipeline
- Mean-time to production went from 2.5hrs (monolith spinnaker pipeline, when the pipeline is not blocked!) to just around 1 hour in our own CI/CD pipeline.

All these improved KPIs translated into much faster and safer project deliveries and engineering time savings.



# Present

After the MVP fellow engineers continued to work and expanded the Discounts service capabilities. At the moment of posting, the discounts service has evolved to support several types of discounts, and new ones are still being migrated and created.



## References

- [Microservices patterns — Chris Richardson](https://microservices.io/book)
- [Monolith to microservices — Sam Newman](https://samnewman.io/books/monolith-to-microservices/)
- [Release It — Michael Nygard](https://pragprog.com/titles/mnee2/release-it-second-edition/)
- [AWS Aurora Master-Slave architecture](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Replication.html)
- [Distributed locks — Martin Kleppmann](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
- [Saga pattern](https://microservices.io/patterns/data/saga.html)
- [Software Architecture The Hard Parts — Neal Ford, Mark Richards](https://learning.oreilly.com/library/view/software-architecture-the/9781492086888/)
