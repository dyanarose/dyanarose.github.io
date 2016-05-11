---
layout: post
title: "Streaming Event Processing"
date: 2016-05-11 20:27:27 +0100
comments: false
categories: 
published: false
---
Comparing distributed Streaming Event Processing engines

Scenario
- Click Stream Event Processing
- Context: 
    - Composite: 
        - Event Interval with Interval taken from website settings
        - Segmentation Context on website and IP
        - Initiator Policy to extend the interval when new Initiating Event is received
- Producer: Click Stream
- Consumers: 
    - Sale Made
    - Timeout
    - Unknown website
    - Unknown event
- Event Producing Agents:
    - Website Enrichment
    - Sale Matcher
    - Timeout Pattern Matcher
    - Session Aggregator
