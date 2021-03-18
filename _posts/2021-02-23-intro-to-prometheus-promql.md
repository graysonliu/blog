---
layout: post
title: "Intro to Prometheus & PromQL"
categories: [prometheus, server, devops]
comments: true
hide: true
---

This post is mainly based on [documentations of Prometheus](https://prometheus.io/docs/introduction/overview/) and [an excellent introduction article](https://yuerblog.cc/2019/01/03/prometheus-client-usage-and-principle/) in Chinese.

## What is a Prometheus metric?

A Prometheus metric consists of a series of `float64` values, with each value associated to a timestamp. Those timestamps in a metric form a time series. A time series is identified by the metric name and the combination of labels. A different metric name, or a different combination of labels, indicates a different time series.

For example, `http_requests_total{method="GET", handler="/info"}` is a metric that records the total number of requests to endpoint `/info` with HTTP method `GET` received by the server, while `http_requests_total{method="POST", handler="/info"}` cares about method `POST` for the same endpoint. Those two represents two different time series because they have different combination of labels, although the metric names are the same.

## Metric Types

### Counter


