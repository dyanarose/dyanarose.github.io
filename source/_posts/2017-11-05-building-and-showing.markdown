---
layout: post
title: "Building and Showing"
date: 2017-11-05 10:57:57 +0000
comments: false
published: true
categories: 
---

On and off I've been working on a site that displays candlestick charts using data from the Guild Wars 2 trading post, but I've never publicly linked to it. 

Part of the reason is that it's not a finished product and I'm not a web designer. But why should that stop me, eh? It's already been an interesting product involving:
 - a service retrieving the data from the GW2 API
 - optimising sql to calculate the candlesticks quickly
 - an RTL service that takes old data and moves to a better long term storage format and then populates a cache
 - a website which must merge cached and uncached data before returning the candlesticks back to the caller.

It didn't start out with 2 services, a website, and a cache though. It started with a program, written in Go that would call an endpoint once an hour and then save the results in a sqlite database.

At that time I didn't know what I wanted to do with the data, I just knew I wanted to do *something*.

Eventually, that something became me answering the question 'what questions do I want the answers to?' As it turns out, what I want is to answer yet another question, 'should I buy, sell, or make this item, and should I do it now.' Now this question I could answer! But my tools at the time would become problematic.

I chose to display historical data using candlestick charts, as I like the view of how prices are moving in a given time period. Calculating open, close, min and max using sqlite proved to be an interesting problem. It was possible though, with some sub-queries, and with some tradeoffs. For example I could only get results for one item at a time, and the larger the dataset, the slower the calculation got. But, most importantly, it gave me a place to start. And it worked! 

Once the requirements started shaking out, infrastructure changes became frequent. I was inside the AWS free tier, but as the sqlite file grew, I started to get worried about EBS storage and keeping things free. So my architecture had to change and I moved to using RDS and PostgreSQL.

Then the fact that the data, once inserted into PostgreSQL, was effectivley dead, started grating on me, and also filling up my free tier allotments of space in RDS.

So I brought in an RTL process to take the dead data out of PostgreSQL, store it more efficiently, and use it to populate a cache.

And then the cache grew too fast and I started seeing evictions. But new problem means new solution. I needed a better way (or even a way) of compressing the data going into the cache. I've worked with Protobuf before so after a bit of a search around to see if Avro would be an immediate better fit, I decided to go with compression via Protobuf. I also reworked the structure of the stored data, because once it was compressed, the keys were still a major source of bloat. And that worked beautifully. I went from being able to store a few months of data to at few years.

And that's where things stand right now. The UI isn't much to talk about but it's clean (sparse some may say) and it's pretty zippy. Which is a long way from the minutes it used to take to load only a single month of data.

http://www.gw2roar.com

I'm going to continue to talk more about the choices I made, and continue to make, and why I had to make them with this project. For example, the significant work around keeping both cache and database sizes sane, automating the ETL process, automating deployments, and anything that comes next.
