---
title: Customer Segmentation at Glovo - Part 1
excerpt: Customer Segmentation at Glovo - Introduction
last_modified_at: 2022-05-15
categories:
  - Audience targeting, microservices, big data
tags:
  - Audience targeting, microservices, big data
toc: true
toc_label: Customer Segmentation - Introduction
toc_icon: code
toc_sticky: true
---

Introduction to [Glovo](https://about.glovoapp.com/):
> Glovo is a Spanish tech company that provides innovative solutions to connect customers, businesses and couriers to provide delivery services, and it is the fastest-growing multicategory player in Europe, Western Asia and Africa.

# Part 1: What is segmentation and why do we need it?

{:refdef: style="text-align: center;"}
![system design book](/assets/images/audience-targeting.jpeg){:height="200px" width="200px"}
{: refdef}

## Business context


Glovo has discount campaigns of different kinds. BeforeBefore this project, these campaigns were targeting the whole customer base, regardless of each customer's behavior and preference. For Glovo and its partners, this was highly inefficient in terms of budget and [ROI](https://www.investopedia.com/terms/r/returnoninvestment.asp#:~:text=Return%20on%20investment%20(ROI)%20is,relative%20to%20the%20investment%27s%20cost.), limiting customer acquisition and retention, and overall customer experience. We didn't have a system capable of classifying the most appropriate customers for each kind of discount campaign: 
- users who never ordered from a specific store
- users who haven't ordered in the last month
- users who order recurrently

We started first by providing a definition of this new concept, which we called "Segmentation",  and the new [bounded context](https://martinfowler.com/bliki/BoundedContext.html) was called "Segments". Its responsibility was to "categorize customers based on different traits such as their profile, their order history, [MParticle](https://www.mparticle.com/?utm_source=google&utm_medium=paid-search&utm_campaign=europe-english-google-paid-search-all-brand&utm_term=mparticle&utm_content=brand-mparticle&persona=brand&gclid=Cj0KCQjwr-SSBhC9ARIsANhzu14iN4DnVLGer1sDd3GN2DGH1Y3lb2fHgezxAacIAXxTDCD4qwU0G4EaAkL8EALw_wcB) analytic events, machine learning models, among others".



## Scope for an MVP

We had the boundaries of our system established, and next we had to prioritize among the multiple stakeholders interested in different segment types. We needed a system capable of supporting most of their use cases, so we had to design it in the right way. Our first segment types were these:
- New customers with 0 orders in certain stores
- Customers ordering recurrently from some stores
- Inactive customers who used to order in the past but they haven't done so in a certain amount of time

Time to market was a hard constraint, so we committed to delivering a solution in only 3 months covering the first segmentation cases, but also be extensible to support different types of segments for the remaining and future use cases. 


# Multi-variable problem

There were multiple challenges and aspects to consider for this project.


### Time-to-market

One of the main core values for Glovo is Gas, as well our competitors and our partners were in deep need of this feature so we had to act fast and with vision.

### Scaling calculations

We would start with only a few hundred segments, but very soon we should scale to hundreds of thousands. A single segment calculation would involve millions of customers and hundreds of millions of order history items. 

### Fast-growing data volume

Not only the number of segments would grow, but also the customer base, order history, and any other new data source involved. We should be ready to process big data.

### Serving segments with high availability and low latency

The segments would be used in most customer flows to customize the customer's journey, so latency was a very crucial constraint and couldn't be hurt. In other words, we committed to a tight [SLA](https://en.wikipedia.org/wiki/Service-level_agreement), with an affordable latency of p99 <= 50 milliseconds and availability >= 99.99%

### Offline data vs Real-time data

Processing and serving terabytes of data to millions of users was challenging, but having this system working with real-time data was even more difficult. Luckily, not all segments required live data, and they could work just fine with eventually consistent segments, with a few minutes delay for updates. We used this to our advantage and started by tackling the use cases that could be near real-time. This simplified our first version of the system, postponing the implementation of those other segments for later.

### Integration

How would these new customer segments be served to the whole Glovo ecosystem in a standardized way? We aimed for a solution that would be agnostic of the calling domain, but at the same time provide the required flexibility to serve each domain's needs. Also, we wanted to integrate other components in the system that could possibly calculate other types of segments in their own way. For example, some Machine Learning models were already generating some kinds of customer segments, and we should be able to feed their results into our service just as a new type of segment.

### Cost

Cost doesn't come only from marketing campaigns, but also from tech, ranging from infrastructure and maintenance costs to new feature development. The cost of the solution cannot be more expensive than the cost of losing partners or unhappy customers. For this project, we aimed for a solution that was cost-effective yet maintainable, reasonably extensible to cover our main use cases in the cluster, and easily evolvable to support new use cases. This allowed us to shorten the delivery of an MVP, meeting our strict time-to-market constraint.

### Customer UX validation

We used [Feature toggles](https://martinfowler.com/articles/feature-toggles.html) and [AB testing](https://en.wikipedia.org/wiki/A/B_testing) to customize each customer's UX and applicable discounts, compare the Return on Investment (ROI) against non-segmented discount campaigns, assessing also the cost of customer acquisition, the flexibility of defining new configurations, and scalability.


# High-level architecture 

Our segmentations component would serve data to the rest of the Glovo ecosystem like this:

{:refdef: style="text-align: center;"}
![system design book](/assets/images/high-level.jpeg)
{: refdef}

We proceeded to assess the nonfunctional requirements, and we quickly realized that the segment's calculation flows were pretty different from the serving data, especially in terms of data volume and required latency. This led us to split calculations and reads into two different components on the architectural level: the ** Segments Core **, responsible for calculating the segments we needed, capable of processing vast and diverse types of data; the ** Segment Store **, responsible for serving the calculated segments to the rest of the system, with high availability and low latency. Having these two separate components would also allow us to choose the appropriate technologies for each task.

If you are interested to know what the architecture looks like check part 2 "Segmentation service zoom in"!
