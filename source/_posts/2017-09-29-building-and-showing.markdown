---
layout: post
title: "Building and Showing"
date: 2017-09-29 10:57:57 +0100
comments: false
published: false
categories: 
---

On and off I've been working on a site that displays candlestick charts using data from the Guild Wars 2 trading post, but I've never publicly linked to it. 

Part of the reason is that it's not a finished product and I'm not a web designer. But why should that stop me, eh? Even in the state it's in it has been an interesting product involving:
 - a service retrieving the data from the GW2 API
 - using window functions to calculate open, close, min, and max values
 - an RTL service that takes old data and moves to a better long term storage format and then populates a cache
 - a website which must merge cached and uncached data before returning the candlesticks back to the caller.

It didn't start out with 2 services, a website, and a cache though. It started with a program, written in Go that would call an endpoint once an hour and then save the results in a sqlite database.

At that time I didn't know what I wanted to do with the data, I just knew I wanted to do something. So I went with sqlite as a format that I could easily move around and didn't really require any setup. It wasn't until I had decided on candlestick analysis that I had my first problem. 

Calculating open, close, min and max using sqlite proved to be an interesting problem. It was possible, with some interesting sub-queries, but I was only able to calculate the results one item at a time, and the larger the dataset, the slower the calculation got. But it worked. 

For the infrastructure, I was inside the AWS free tier, but as the sqlite file grew, I started to get worried about EBS storage and keeping things free. So my architecture had to change and I moved to using RDS and PostgreSQL.

Then the fact that the data, once inserted into PostgreSQL, was effectivley dead, started grating on me, and also filling up my free tier allotments of space in RDS.

So in comes an RTL process to take the dead data out of PostgreSQL, store it more efficiently, and use it to populate a cache.

And then the cache started growing and I started seeing evictions. But new problem means new solution. I needed a better way (or even a way) of compressing the data going into the cache. I've worked with Protobuf before so after a bit of a search around to see if Avro would be an immediate better fit, I decided to go with compression via Protobuf and also reworked the structure of the stored data as keys had now become a major source of bloat. And that worked beautifully. I went from being able to store a few months of data to at few years.

And that's where things stand right now. The UI isn't much to talk about, a search function and a stock from Highcharts. But it's pretty zippy now, a long way from the minutes it used to take to load only a months worth of data.

http://gw2technicalanalysis.dyanarose.co.uk (still don't have a good name for this)

I'm going to continue to talk more about the choices I made, and continue to make, and why I had to make them with this project. For example, the significant work around keeping both cache and database sizes sane, automating the ETL process, automating deployments, and anything that comes next (I'd like to get back onto using ECS for example, which I stopped using when how I wanted to store the old sqlite file wasn't very compatible with the way ECS wants to be set up.)
