---
layout: blog
title: Tail Latency
---

<span class="hidden-text"># Created: 2021-06-27; Modified: 2021-06-27</span>

## Observing Tail

Check out Jil Tene's talk on measuring latency [here](https://www.infoq.com/presentations/latency-response-time/) (2016).

For serious quantification, there are some common QoS metrics for tail latency:
- p95, p99, ...
- p99/avg
- standard deviation
- [norm](https://en.wikipedia.org/wiki/Norm_(mathematics))

## Tails in Service Time

## Tails in Queueing

## Tails in System

Here "System" refers to a composite service built on top of several black box service units. Like Lego system, there are limited ways to combine several units, even though they can sum up quite differently.

### Pipeline

Each task is processed by multiple units in sequence.

### Parallel

Each task is divided into multiple sub-tasks which gets processed in parallel.

<!-- On tail latency:

If a request is too slow:
- Cancel it
- Re-issue it
- Defer it

Pipelined queueing

Parallel queueing
 -->