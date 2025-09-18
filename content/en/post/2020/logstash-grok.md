---
layout: post
title: logstash grok Configuration Patterns
category: ELK
date: 2018-05-20
tags:
- ELK
- logstash
---
# logstash grok Configuration Patterns

## logstash.conf

Configure `grok { match => ... }` to split logs into indexed fields (matches are essentially regex patterns).

Raw log:

```log
2018-04-13 16:03:49.822 INFO  o.n.p.j.c.XXXXX - Star Calculator
```

grok match:

```json
match => { "message" => "%{DATA:log_date} %{TIME:log_localtime} %{WORD:log_type} %{JAVAFILE:log_file} - %{GREEDYDATA:log_content}"}
```

Parsed fields:

```json
{
  "log_date": [["2018-04-13"]],
  "log_localtime": [["16:03:49.822"]],
  "HOUR": [["16"]],
  "MINUTE": [["03"]],
  "SECOND": [["49.822"]],
  "log_type": [["INFO"]],
  "log_file": [["o.n.p.j.c.XXXX"]],
  "log_content": [["Star Calculator"]]
}
```

All extracted fields are ES mapping indices and can be queried.

Test patterns at: http://grokdebug.herokuapp.com/ and view available patterns at http://grokdebug.herokuapp.com/patterns

We use a config like `/logstash/logstash-k8s.conf`.

## Q: How to force data types in mapping?

A: `grok` is regex-based; defaults are strings. You can cast numeric fields inline, e.g. `%{NUMBER:response_time:int}`.

Full example:

```json
match => {
  "message" => "%{DATA:log_date} %{TIME:log_localtime} %{WORD:log_type}  %{JAVAFILE:log_file} - %{WORD:method} %{URIPATHPARAM:uri} %{NUMBER:status:int} %{NUMBER:size:int} %{NUMBER:response_time:int}"
}
```

## Q: Store indices by date?

A: In `output`, set index format, e.g. `index => "k8s-%{+YYYY.MM.dd}"`.

Full output:

```json
output {
  elasticsearch {
    hosts => "${ES_URL}"
    manage_template => false
    index => "k8s-%{+YYYY.MM.dd}"
  }
}
```

Full logstash.conf:

```conf
input {
  beats {
    host => "0.0.0.0"
    port => 5043
  }
}
filter {
  if [type] == "kube-logs" {
    mutate { rename => ["log", "message"] }
    date {
      match => ["time", "ISO8601"]
      remove_field => ["time"]
    }
    grok {
      match => {
        "source" => "/var/log/containers/%{DATA:pod_name}_%{DATA:namespace}_%{GREEDYDATA:container_name}-%{DATA:container_id}.log"
      }
      match => {
        "message" => "%{DATA:log_date} %{TIME:log_localtime} %{WORD:log_type}  %{JAVAFILE:log_file} - %{WORD:method} %{URIPATHPARAM:uri} %{NUMBER:status:int} %{NUMBER:size:int} %{NUMBER:response_time:int}"
      }
      remove_field => ["source"]
      break_on_match => false
    }
  }
}
output {
  elasticsearch {
    hosts => "${ES_URL}"
    manage_template => false
    index => "k8s-%{+YYYY.MM.dd}"
  }
}
```

