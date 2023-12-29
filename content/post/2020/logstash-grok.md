---
layout: post
title: logstash grok配置规则
category: ELK
date: 2018-05-20
tags:
- ELK
- logstash
---
# logstash grok配置规则

## [logstash.conf](https://www.elastic.co/guide/en/beats/metricbeat/current/logstash-output.html)

这里主要需要配置grok match,把日志信息切分成索引数据(match本质是一个正则匹配)

日志原文:

```log
2018-04-13 16:03:49.822 INFO  o.n.p.j.c.XXXXX - Star Calculator
```

grok match:

```json
 match => { "message" => "%{DATA:log_date} %{TIME:log_localtime} %{WORD:log_type} %{JAVAFILE:log_file} - %{GREEDYDATA:log_content}"}
```

切出来的数据

``` json
{
  "log_date": [
    [
      "2018-04-13"
    ]
  ],
  "log_localtime": [
    [
      "16:03:49.822"
    ]
  ],
  "HOUR": [
    [
      "16"
    ]
  ],
  "MINUTE": [
    [
      "03"
    ]
  ],
  "SECOND": [
    [
      "49.822"
    ]
  ],
  "log_type": [
    [
      "INFO"
    ]
  ],
  "log_file": [
    [
      "o.n.p.j.c.XXXX"
    ]
  ],
  "log_content": [
    [
      "Star Calculator"
    ]
  ]
}

```

上面所有切出来的field都是es中mapping index,都可以在用来做条件查询.

[grokdebug.herokuapp.com](http://grokdebug.herokuapp.com/ )里面可以做测试.

[grokdebug.herokuapp.com/patterns](http://grokdebug.herokuapp.com/patterns) 所有可用的patterns都可以在这里查到.

现在我们在用的配置见/logstash/logstash-k8s.conf

## Q: 需要指定mapping index的数据类型怎么办?

A: grok match本质是一个正则匹配,默认出来的数据都是String.有些时候我们知道某个值其实是个数据类型,这时候可以直接指定数据类型. 不过match中仅支持直接转换成int ,float,语法是 %{NUMBER:response_time:int}
完整配置:

``` json
match => {
            "message" => "%{DATA:log_date} %{TIME:log_localtime} %{WORD:log_type}  %{JAVAFILE:log_file} - %{WORD:method} %{URIPATHPARAM:uri} %{NUMBER:status:int} %{NUMBER:size:int} %{NUMBER:response_time:int}"}
```

## Q: 索引文件想需要按日期分别存放,怎么办?

A:  out中指定index格式,如 index=> "k8s-%{+YYYY.MM.dd}"

完整out如下:

``` json
output {
    elasticsearch {
      hosts => "${ES_URL}"
      manage_template => false
      index => "k8s-%{+YYYY.MM.dd}"
      }
  }

```

完整logstash.conf

```conf
input {
    beats {
      host => "0.0.0.0"
      port => 5043
    }
  }
  filter {
    if [type] == "kube-logs" {
      mutate {
        rename => ["log", "message"]
      }
      date {
        match => ["time", "ISO8601"]
        remove_field => ["time"]
      }
      grok {
          match => {
            "source" => "/var/log/containers/%{DATA:pod_name}_%{DATA:namespace}_%{GREEDYDATA:container_name}-%{DATA:container_id}.log"}
          match => {
            "message" => "%{DATA:log_date} %{TIME:log_localtime} %{WORD:log_type}  %{JAVAFILE:log_file} - %{WORD:method} %{URIPATHPARAM:uri} %{NUMBER:status:int} %{NUMBER:size:int} %{NUMBER:response_time:int}"}
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