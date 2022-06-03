---
title: Learnings from team rotations
excerpt: My learnings from my recent team rotation
last_modified_at: 2021-10-12
categories:
  - Modularization
tags:
  - Modularization
toc: true
toc_label: Team rotation learnings
toc_icon: code
toc_sticky: true
---

Introduction to [Glovo](https://about.glovoapp.com/):
> Glovo is a Spanish tech company that provides innovative solutions to connect customers, businesses and couriers to provide delivery services, and it is the fastest-growing multicategory player in Europe, Western Asia and Africa.



# Team rotations at Glovo

This post is a replica of my [original post](https://medium.com/glovo-engineering/erasmus-like-engineering-rotations-a-tool-for-knowledge-sharing-78ee01893ccf) in Glovo's Medium blog.

In this post we describe the experience of a team rotation, how it went and the lessons we learned.



# Introduction

I rotated to the Organic Growth team for three sprints to help cover an engineer that was about to go on paternity leave. Coming from a year of working in Promotions and Discounts domains it was an opportunity to try out something different. The Organic team owns the “domain responsible to acquire new customers organically”, things like Search Engine Optimisation (SEO).

# The tech stack

Before jumping to the rotation itself, the tech stack and way of working at Glovo: we use a large set of technologies, yet the most widely used stack in the Backend is Java 8, Spring framework, Gradle, and AWS Aurora — MySQL and several others. Our teams are agile and multidisciplinary, consisting mostly of backenders, frontenders, mobile engineers, data analysts, BI engineers, an Engineering Manager and a Product Manager.

Although Organic was a rather new team — just 6 months old, I found well-grounded engineers and managers with a solid understanding of Glovo’s ecosystem (being at Glovo for a while already, I didn’t expect any less). I was onboarded very nicely to the team, so I could hit the ground running. Jumping to their Scrum ceremonies, I was surprised to find how closely the team worked, how fluent the communication was, and how efficient the participation/collaboration across projects and stacks was (Frontenders contributing in Backend tasks analysis and estimations and vice versa). Plus the way of calculating story points and estimating the capacity of the team for the Scrum Sprint per stack was very precise. I found these exchanges to be very helpful to understand what the rest of the engineers were working on.

# The project

The project I joined was about publishing our partner’s information to Google Maps via SFTP, in order to enable placing orders from our Glovo app. First I reviewed the requirements, the proposed solution and architecture. It seemed a rather simple task: gather data from the DB, transform it and push it to the external system. Yet at Glovo few things are that easy. We had to process over 100K entries in one go (otherwise if disconnected google would discard the partially uploaded file), minimising the risks of impacting our huge and sensitive monolithic DB with long-running transactions, and watching out for the JVM memory footprint during the process. Also, we had to consider the monolith CI/CD times of up to 3 hs from merge to production (partly caused by the growth rate of the data, new logic added every hour and an ongoing microservices migration). And as if these were not enough, we had a file size restriction of 200MB max on Google’s api as well.

# Challenges and solutions

Most of these challenges were easily overcome by decoupling the producer and consumer, using batching and [DTO projections](https://www.baeldung.com/spring-data-jpa-projections) when reading from the DB, pushing the data in chunks, covering new flows behind [feature toggles](https://martinfowler.com/articles/feature-toggles.html), and making sure we had proper metrics and monitoring (for which we use [Datadog](https://www.datadoghq.com/blog/monitoring-101-collecting-data/) with Infra as code by [Terraform](https://www.terraform.io/)). About the large file size and execution times, gzipping the stream brought a big improvement: the time of the task went from 54 min to 16 min, and the size of the file from 60MB to just 7MB.

## The modular monolith

{:refdef: style="text-align: center;"}
![system design book](/assets/images/modular-monolith.jpeg)
{: refdef}

The long execution times of the test suites in the monolith (with its heavy Spring context loading times, constantly growing) were not exciting at all, especially after coming from months of working on the Discounts microservice. Although this new project was initially proposed as a microservice, it wasn’t big enough to justify a whole new infra setup. So what then? The Organic backenders proposed to use an approach based on [domain-driven design](https://martinfowler.com/tags/domain%20driven%20design.html#:~:text=Domain%2DDriven%20Design%20is%20an,through%20a%20catalog%20of%20patterns.) and [ports and adapters](https://netflixtechblog.com/ready-for-changes-with-hexagonal-architecture-b315ec967749) that had worked pretty well in another project: create the code for the new bounded context in a fully decoupled gradle module, with no dependencies to the core monolith domain. Only a few adapters would live in the monolith module, implementing the corresponding ports from our core module. This way the core domain was simple and with a fast suite of Unit and Integration tests. I found that working in this way was a breath of fresh air when working in the monolith.

# Results

On my last day, we already had our production data being pushed to Google and validated, and we were wrapping up some additional monitoring and a final refactor. Now I’m going back to my original team, glad to have met Organic people and exchanged ideas, and eager to put my learnings into practice in a big new project that’s just starting.


