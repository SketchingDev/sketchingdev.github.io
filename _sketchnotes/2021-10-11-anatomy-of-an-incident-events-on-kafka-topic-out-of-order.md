---
layout: sketchnote
title: "Anatomy of an Incident: Events on Kafka topic out of order"
date: 2021-10-05 00:00:00
thumbnail: /assets/images/sketchnotes/2021-10-11-anatomy-of-an-incident-events-on-kafka-topic-out-of-order/kafka-incident-thumbnail.jpg
image: /assets/images/sketchnotes/2021-10-11-anatomy-of-an-incident-events-on-kafka-topic-out-of-order/kafka-incident.jpg
image_ratio: wide
---

Recently at work a team rewrote their Kafka producer and then noticed unordered events appearing in their topic. I didn't
know how this was possible - given my limited knowledge of Kafka - so drew this sketch based on their explanation, which
helped me understand what was happening...

Here's a written breakdown:

## Overview of how Kafka partitions data

The Producer's Partitioner decides on which partition Kafka should store the message in on the topic. It can decide this based on the message's key, from the producer or simply round-robin.

![](/assets/images/sketchnotes/2021-10-11-anatomy-of-an-incident-events-on-kafka-topic-out-of-order/overview-of-partitioning-data.png)

Kafka guarantees the order of messages within these partitions, but not **across** partitions.

When the service was rewritten a Partitioner from another language was used. Although both decided on partitions based on the message's key, the new one did it differently.

![](/assets/images/sketchnotes/2021-10-11-anatomy-of-an-incident-events-on-kafka-topic-out-of-order/incident.png)

This means that if a consumer re-consumed the topic then the events for specific accounts would be out of order.
