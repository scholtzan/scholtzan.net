+++
title = "My Internship at Mozilla"
date = "2019-08-27"
type = "post"
tags = ["Mozilla", "Open Source", "Internship"]
+++

This summer I was an intern on the Data Platform Team at Mozilla - from beginning of June until the end of August 2019. I was located in the Mountain View office which also happens to be Mozilla's headquarters. It was my first time ever in California and working in the US, so I was especially excited.

The first week was mostly spent on getting to know other interns, my mentor and my team. The majority of the people on the Data Platform Team, including my mentor, worked remotely, so I only had a chance to meet them virtually.


## Projects

Over the duration of my internship I had a chance to work on many different projects. The first project I worked on was about getting Python code running in BigQuery. I already wrote a [blog post](https://scholtzan.net/blog/bigquery-udf-python/) earlier that summarizes what I did and my project results.

The things I learned in my first project also helped me in working on some smaller-scoped issues in the [BigQuery ETL](https://github.com/mozilla/bigquery-etl) projects. Besides making some simplifications for deploying user-defined functions, I also wrote a user-defined functions for decompressing gzipped data.

The second project I took on was related to the development of a real-time data pipeline for aggregating telemetry data sent from Firefox. The goal of this data pipeline is to enable real-time decision making, for example through crash monitoring or by providing real-time insights to data scientists. Collected data is stored in a [Druid](https://github.com/apache/incubator-druid) database and can be queried from there and used for analysis. Since the database is a crucial part of the pipeline it is important to be able to monitor its system health to detect problems early on. For this, I implemented a [plugin for Druid that emits metrics to Stackdriver](https://github.com/scholtzan/stackdriver-emitter). This way it is possible not only to visualize in Stackdriver, for example, the memory usage and CPU load of Druid but also the number of executed queries and how many queries failed to run. 

After finishing the Stackdriver metric emitter plugin, I worked on data ingestion from Google Pub/Sub for Druid. Firefox telemetry events are written to Pub/Sub and should be ingested from there, however by default Druid only has support for data ingestion from Kafka. To add Pub/Sub support, I decided to write a [custom indexing service for Druid](https://github.com/scholtzan/druid-pubsub-indexing-service) that pulls data from Pub/Sub and creates data segments that are written to deep storage. Unfortunately, due to the lack of documentation, writing the service turned out to be quite difficult and resulted in code that is rather hard to maintain for others. As a workaround I decided to extend [Tranquility](https://github.com/druid-io/tranquility) which is basically a service that provides a REST API and makes accessing and sending data in Druid easier. I implemented an [adapter for Pub/Sub into tranquility](https://github.com/scholtzan/tranquility) that pulls events from Pub/Sub and sends them to Druid which will use its default indexing service to store the data.

One of the nice thing about Mozilla is that I can make almost all of the things and projects I have worked on public and open source. So all of the stuff I implemented and developed while working at Mozilla can be found on my [GitHub profile](https://github.com/scholtzan).



## Life at Mozilla

Besides working on very interesting projects with awesome people, one of the best things were the events Mozilla organized for interns or employees in general.

After my second week, Mozilla organized an All Hands in Whistler. This was an awesome opportunity to meet a lot of awesome Mozillians as well as my team in person. I learned a lot about other teams and projects and very much enjoyed the nature and scenery around Whistler.

The university team also organized a variety of events for all of us interns: we visited Alcatraz, had an escape room and a painting event. Additionally, we had a weekly speaker series where senior leaders talk about what they do at Mozilla and answer questions. 

Other awesome things were the free lunch that was offered daily, a lot of games that were lying around the office, a ballpit (I witnessed 3 people losing their phone in there during my internship) and a free gym. 

## Conclusion

I learned so many things about data pipelines and data engineering during my time at Mozilla. I am very grateful for my mentor and later co-mentor for their support and for letting me explore so many different projects. 

All the people I have met were super friendly, very open and always interested in what I am working on. The past 12 weeks went by really fast and I am very sad to leave. But I hope I will be back one day.

