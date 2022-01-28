# 第一套

0.环境

```
cluster1: node1,node2,node3,kibana1
cluster2: node4,kibana2
不要在集群中添加remote_cluster角色
```

1.Food-ingredient, 包含“cake mix” 的 manufacturer，name，brand，使用em标签来高亮name字段 

````
# 数据
PUT  food-ingredient/_bulk
{"index": {"_id":1}}
{"manufacturer": "cake mix apple banana","name":"mix cake","brand":"xxx cake"}
{"index": {"_id":2}}
{"manufacturer": "apple banana","name":"cake mix","brand":"xux fruit"}
{"index": {"_id":3}}
{"manufacturer": "apple banana mix","name":"fruit","brand":"sds mix fruit"}
{"index": {"_id":4}}
{"manufacturer": "apple cake","name":"fruit","brand":"yyy cake"}

# 错误答案，注意这道题不能用multi_match来解，只要是短语默认使用match_phrase
GET food-ingredient/_search
{
  "query": {
    "multi_match": {
      "query": "cake mix",
      "fields": ["manufacturer","name","brand"]
    }
  },
  "highlight": {
    "fields": {
      "name" : { "pre_tags" : ["<em>"], "post_tags" : ["</em>"] }
    }
  }
}
# 正确答案：为了方便考试模拟，将部分答案放到最后，做完可以参考。部分需要特殊说明的答案直接放在题目后了
````

2.CCR ：从cluster2中将索引ccr_test复制到cluster1中

```
# 环境：
# 这里模拟时增加了难度，在cluster1没有配置security，我考试时是两个集群都配置好的，但是以防考security，所以模拟一下配置
	集群cluster1: 包含节点node1,node2,node3，集群cluster2：包含节点node4；
	cluster2中已经配置好security,并且部署了证书。cluster1中没有配置security
	cluster2中有索引ccr_test
# 数据
PUT ccr_test/_doc/1
{"ccr":"you did it"}
# 答案
# 1、开启远程集群映射
PUT _cluster/settings
{
  "persistent": {
    "cluster": {
      "remote": {
        "cluster_one": {
          "seeds": [
            "172.16.188.10:9300",
            "172.16.188.11:9300",
            "172.16.188.12:9300"
          ]
        },
        "cluster_two": {
          "seeds": [
            "172.16.188.8:9300"
          ]
        }
      }
    }
  }
}
# 执行以下命令，会发现报错说证书的问题，这是应为cluster1没有开启security，且证书需要和cluster2保持一致
PUT /ccr_test/_ccr/follow?wait_for_active_shards=1
{
  "remote_cluster" : "cluster_two",
  "leader_index" : "ccr_test"
}
# 复制证书到cluster1
scp config/elastic-certificates.p12 root@172.16.188.10:/var/local/elasticsearch/config
scp config/elastic-certificates.p12 root@172.16.188.11:/var/local/elasticsearch/config
scp config/elastic-certificates.p12 root@172.16.188.12:/var/local/elasticsearch/config
# 
su
chown -R elastic:elastic config/elastic-certificates.p12
su elastic
vim config/elasticsearch.yml
# 
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate 
xpack.security.transport.ssl.client_authentication: required
xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: elastic-certificates.p12
#
./bin/elasticsearch -d
./bin/elasticsearch-setup-passwords interactive
# 将密码设置为elastic的登陆密码
# 
PUT /ccr_test/_ccr/follow?wait_for_active_shards=1
{
  "remote_cluster" : "cluster_two",
  "leader_index" : "ccr_test"
}
GET ccr_test/_search
```



3.stock price，runtime field里定义一个字段difference，用close-price减去open-price，然后根据这个字段进行桶聚合，分三个桶：<-5，-5到5，>5 

```
# 数据
PUT stock_price/_bulk
{"index":{"_id":1}}
{"close-price": 20.98,"open-price":10.78}
{"index":{"_id":2}}
{"close-price": 10.00,"open-price":15.78}
{"index":{"_id":3}}
{"close-price": 20.80,"open-price":20.40}
{"index":{"_id":4}}
{"close-price": 20.30,"open-price":21.00}

```



4.地震索引，只要2012年的数据，按月分桶，然后每个桶里对magnitude和depth进行最大值聚合 

```
# 数据
# 注意：根据同学反馈，题目中的format还有可能是MM/dd/yyyy，或者dd/MM/yyyy的形式，具体根据考试实际情况判断，先查询下索引的mappings。练习时建议三种情况都训练一下
PUT earthquakes
{
  "mappings": {
    "properties": {
      "pt": {
        "type": "date",
        "format": "yyyy/MM/dd HH:mm:ss"
      }
    }
  }
}

POST earthquakes/_bulk
{"index":{"_id":1}}
{"pt":"2012/01/01 17:00:00", "magnitude":5, "depth":5}
{"index":{"_id":2}}
{"pt":"2012/01/01 20:00:00", "magnitude":5, "depth":1}
{"index":{"_id":3}}
{"pt":"2012/02/01 17:00:00", "magnitude":1, "depth":1}
{"index":{"_id":3}}
{"pt":"2012/02/20 17:00:00", "magnitude":3, "depth":5}
{"index":{"_id":4}}
{"pt":"2012/11/01 17:00:00", "magnitude":5, "depth":4}
{"index":{"_id":5}}
{"pt":"2012/11/01 17:00:00", "magnitude":7, "depth":7}
{"index":{"_id":6}}
{"pt":"2012/11/01 17:00:00", "magnitude":9, "depth":8}
{"index":{"_id":7}}
{"pt":"2012/01/01 17:00:00", "magnitude":8, "depth":5}
{"index":{"_id":8}}
{"pt":"2016/01/01 20:00:00", "magnitude":11, "depth":12}
{"index":{"_id":9}}
{"pt":"2012/02/01 17:00:00", "magnitude":5, "depth":5}
{"index":{"_id":10}}
{"pt":"2016/02/20 17:00:00", "magnitude":6, "depth":5}
{"index":{"_id":11}}
{"pt":"2012/11/01 17:00:00", "magnitude":7, "depth":7}
{"index":{"_id":12}}
{"pt":"2016/11/01 17:00:00", "magnitude":2, "depth":1}
{"index":{"_id":13}}
{"pt":"2012/11/01 17:00:00", "magnitude":3, "depth":3}
{"index":{"_id":14}}
{"pt":"2012/11/01 17:00:00", "magnitude":6, "depth":5}
{"index":{"_id":15}}
{"pt":"2012/11/01 17:00:00", "magnitude":7, "depth":7}

```



5.Yoo-hoo 和 yoohoo作为查询条件，将yoohoo索引reindex到yoohoo_reindex后使得他们查询评分一样

```
# 数据
PUT yoohoo/_bulk
{"index":{"_id":1}}
{"title": "Yoo-hoo used to be hot"}
{"index":{"_id":2}}
{"title": "Yoohoo used to be hot"}
{"index":{"_id":3}}
{"title": "I often login to yoohoo!"}

```



6.搜索模板，在item里搜索参数query_item，在timestamp里搜索range，参数为start_date和end_date，而且status字段为active，如果没有提供end_date, 默认为now 

创建一个搜索模版item_search_template，实现如下要求

​	item为参数query_item；timestamp从start_date到end_date范围，status为"active",如果end_date没有传值默认为now

```
# 数据
PUT item/_bulk
{"index":{"_id":1}}
{"item": "apple banana","timestamp":"2012-01-01T17:00:00","status":"active"}
{"index":{"_id":2}}
{"item": "apple","timestamp":"2013-01-01T17:00:00","status":"close"}
{"index":{"_id":3}}
{"item": "banana","timestamp":"2013-12-01T17:00:00","status":"active"}
{"index":{"_id":4}}
{"item": "apple banana","timestamp":"2013-12-01T17:00:00","status":"active"}
{"index":{"_id":5}}
{"item": "banana","timestamp":"2014-01-01T17:00:00","status":"active"}

# 错误答案，params并不能设置默认值，如果要设置参数默认值，使用{{my-var}}{{^my-var}}default value{{/my-var}}格式
PUT _scripts/item_search_template
{
  "script": {
    "lang": "mustache",
    "source": {
      "query": {
        "bool": {
          "must": [
            {
              "match": {
                "status": "active"
              }
            },
            {
              "term": {
                "item": "{{query_term}}"
              }
            },
            {
              "range": {
                "timestamp": {
                  "gte": "{{start_date}}",
                  "lt": "{{end_date}}"
                }
              }
            }
          ]
        }
      }
    }
  },
  "params": {
    "end_date": "now/d"
  }
}
```



7.索引movies中查询title包含me或者my，如果tags字段（数组）包含“romantic comedy”，那么提高权重

```
# 数据
PUT movies/_bulk
{"index":{"_id":1}}
{"title": "you are my love","tags":["romantic comedy","Action"]}
{"index":{"_id":2}}
{"title": "you and me","tags":["Action","Tragedy"]}
{"index":{"_id":3}}
{"title": "my life","tags":["Tragedy","Tragedy"]}
{"index":{"_id":4}}
{"title": "We are always together","tags":["romantic comedy","Tragedy"]}
{"index":{"_id":5}}
{"title": "me and my dog","tags":["romantic comedy","Action"]}
{"index":{"_id":6}}
{"title": "me and my cat","tags":["comedy","Action"]}

# 注意这里要使用match_phrase，查询短语默认使用match_phrase
```



8.创建数据流。要求：

​	· 在mappings里增加4个字段，host.name, error.message, timestamp, tags (数组）, 而且host.name 和tags是keyword only，error.message是text only并用standard analyzer。 	

​	· 根据这个模板创建一个数据流，命名为 mymetrics-exam.prod，索引pattern是`mymetrics-*.* `

​	· 设置主分片数为1个，副分片数为2个

​	· 给出了一条数据，把这条数据插入该数据流 

```
# 数据
{
  "timestamp": "2021-01-01",
  "host": {
    "name": "xx"
  },
  "error": {
    "message": "xx"
  },
  "tags": [
    "ss",
    "ss"
  ]
} 
PS: 有同学在做这道题时创建完数据流出现2个副节点未分配的情况，可能是隐藏考点：需要修复集群问题，这里无法模拟出集群环境故暂时未考虑集群问题。
```



9.题目给了一条数据，里面有个字段features，features是个数组，里面每个元素是{‘type’: ‘xxx’, ‘value’: ‘yyy’}，然后题目又给了一条查询语句并且告诉你改语句应该查不出任何结果，但是实际查出来了。要求创建一个新的索引features_reindex，reindex到新索引上，然后写一条查询语句，查不出结果。就是要把features换成nested类型，然后写一条查询语句来查询nested的内容。要求查询的数据{‘type’: ‘storage’, ‘vaue’: ’12’} 

```
# 数据
PUT features/_bulk
{"index":{"_id":1}}
{"features": [{"type":"storage","value":"12"},{"type":"aaa","value":"bbb"},{"type":"ccc","value":"ddd"}]}
{"index":{"_id":2}}
{"features": [{"type":"storage","value":"13"},{"type":"aaa","value":"bbb"},{"type":"ccc","value":"ddd"}]}

```



10.给了两个索引，earthquakes_data和magnityde_type_desc。这两个索引里有一个共同的字段，并且是关联字段 magnitude_type，第二个索引里的第二个字段是desc，用来描述地震类型。要求创建一个新的索引earthquakes_data_new，里面包含了earthquakes_data里面的全部数据，并且每一条数据都要根据地震类型添加一个新的字段，就是desc。

```json
POST earthquakes_data/_bulk
{"index":{}}
{"country":"Title A","magnityde":"yyy","magnityde_type":"aaa"}
{"index":{}}
{"country":"Title B","magnityde":"yyy","magnityde_type":"bbb"}
{"index":{}}
{"country":"Title D","magnityde":"yyy","magnityde_type":"ddd"}
{"index":{}}
{"country":"Title E","magnityde":"yyy","magnityde_type":"eee"}
{"index":{}}
{"country":"Title E","magnityde":"yyy","magnityde_type":"hhh"}

POST magnityde_type_desc/_bulk
{"index":{}}
{"magnityde_type":"aaa","desc":"this is aaa"}
{"index":{}}
{"magnityde_type":"bbb","desc":"this is bbb"}
{"index":{}}
{"magnityde_type":"ccc","desc":"this is ccc"}
{"index":{}}
{"magnityde_type":"eee"}
{"index":{}}
{"magnityde_type":"hhh","desc":null}

```

### 答案

```
# 1.Food-ingredient, 包含“cake mix” 的 manufacturer，name，brand，使用em标签来高亮name字段 

GET food-ingredient/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "match_phrase": {
            "manufacturer": "cake mix"
          }
        },
        {
          "match_phrase": {
            "name": "cake mix"
          }
        },
        {
          "match_phrase": {
            "brand": "cake mix"
          }
        }
      ]
    }
  },
  "highlight": {
    "fields": {
      "name" : { "pre_tags" : ["<em>"], "post_tags" : ["</em>"] }
    }
  }
}

# 2 CCR ：从cluster2中将索引ccr_test复制到cluster1中
PUT _cluster/settings
{
  "persistent": {
    "cluster": {
      "remote": {
        "cluster_one": {
          "seeds": [
            "172.16.188.10:9300",
            "172.16.188.11:9300",
            "172.16.188.12:9300"
          ]
        },
        "cluster_two": {
          "seeds": [
            "172.16.188.8:9300"
          ]
        }
      }
    }
  }
}
# 
DELETE ccr_test
GET ccr_test/_search
PUT /ccr_test/_ccr/follow?wait_for_active_shards=1
{
  "remote_cluster" : "cluster_two",
  "leader_index" : "ccr_test"
}
# 3 stock price，runtime field里定义一个字段difference，用close-price减去open-price，然后根据这个字段进行桶聚合，分三个桶：<-5，-5到5，>5 
GET stock_price/_search
{
  "size": 0, 
  "runtime_mappings": {
    "difference": {
      "type": "double",
      "script": {
        "source": "emit(doc['close-price'].value - doc['open-price'].value)"
      }
    }
  },
  "aggs": {
    "diff_bucket": {
      "range": {
        "field": "difference",
        "ranges": [
          { 
            "to": -5
          },
          {
            "from": -5, 
            "to": 5
          },
          {
            "from": 5
          }
        ]
      }
    }
  }
}
# 4 地震索引，只要2012年的数据，按月分桶，然后每个桶里对magnitude和depth进行最大值聚合
GET earthquakes/_search
{
  "size": 0,
  "query": {
    "range": {
      "pt": {
        "gte": "2012/01/01 00:00:00",
        "lte": "2012/12/31 23:59:59"
      }
    }
  },
  "aggs": {
    "month_bucket": {
      "date_histogram": {
        "field": "pt",
        "calendar_interval": "month"
      },
      "aggs": {
        "max_mag": {
          "max": {
            "field": "magnitude"
          }
        },
        "max_depth": {
          "max": {
            "field": "depth"
          }
        }
      }
    }
  }
}
# 5 Yoo-hoo 和 yoohoo作为查询条件，将yoohoo索引reindex到yoohoo_reindex后使得他们查询评分一样
GET yoohoo/_search
DELETE yoohoo_reindex
PUT yoohoo_reindex
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_analyzer": {
          "tokenizer": "standard",
          "char_filter": [
            "my_char_filter"
            ],
          "filter": [
            "lowercase"
            ]
        }
      },
      "char_filter": {
        "my_char_filter": {
          "type": "mapping",
          "mappings": [
            "-=>"
            ]
        }
      }
    }
  }, 
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "my_analyzer"
      }
    }
  }
}
POST _reindex
{
  "source": {
    "index": "yoohoo"
  },
  "dest": {
    "index": "yoohoo_reindex"
  }
}
GET yoohoo_reindex/_search
{
  "query": {
    "match": {
      "title": "Yoo-hoo"
    }
  }
}
GET yoohoo_reindex/_search
{
  "query": {
    "match": {
      "title": "yoohoo"
    }
  }
}
# 6 搜索模板，在item里搜索参数query_item，在timestamp里搜索range，参数为start_date和end_date，而且status字段为active，如果没有提供end_date, 默认为now 
#创建一个搜索模版item_search_template，实现如下要求
#	item为参数query_item；timestamp从start_date到end_date范围，status为"active,如果end_date没有传值默认为now
GET item/_search
PUT _scripts/my_search_template
{
  "script": {
    "lang": "mustache",
    "source": {
      "query": {
        "bool": {
          "must": [
            {
              "match": {
                "item": "{{query_item}}"
              }
            },
            {
              "range": {
                "timestamp": {
                  "gte": "{{start_date}}",
                  "lte": "{{end_date}}{{^end_date}}now{{/end_date}}"
                }
              }
            },
            {
              "term": {
                "status": "active"
              }
            }
          ]
        }
      }
    }
  }
}

GET item/_search/template
{
  "id": "my_search_template",
  "params": {
    "query_item": "banana",
    "start_date": "2012-01-01",
    "end_date": "2013-01-01"
  }
}

# 7 查询title包含me或者my，如果tags字段（数组）包含“romantic comedy”，那么提高权重
GET movies/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "terms": {
            "title": [
              "me",
              "my"
            ]
          }
        }
      ],
      "should": [
        {
          "match_phrase": {
            "tags": {
              "query": "romantic comedy",
              "boost": 2
            }
          }
        }
      ]
    }
  }
}
# 8 创建数据流。索引pattern是`mymetrics-*.* `, 按要求在mappings里增加4个字段，host.name, error.message, timestamp, tags (数组）, 而且host.name 和tags是keyword only，error.message是text only并用standard analyzer。 然后根据这个模板创建一个数据流，命名为 mymetrics-exam.prod。设置主分片数为1个，副分片数为2个。题目中给出了一条数据，把这条数据插入该数据流 
# 1 创建组件模版，设置mapping，分片数
PUT _component_template/my-settings
{
     "template": {
         "settings": {
               "number_of_replicas": 2,
               "number_of_shards": 1
           }
     }
}
PUT _component_template/my-mappings
{
  "template": {
    "mappings": {
      "properties": {
        "timestamp": {
          "type": "date"
        },
        "error": {
          "properties": {
            "message": {
              "type": "text",
              "analyzer": "standard"
            }
          }
        },
        "host": {
          "properties": {
            "name": {
              "type": "keyword"
            }
          }
        },
        "tags": {
          "type": "keyword"
        }
      }
    }
  }
}
# 2 创建索引模版，索引以mymetrics-*.*进行匹配
PUT _index_template/my_template
{
  "index_patterns": ["mymetrics-*.*"],
  "data_stream": { },
  "composed_of": [ "my-mappings", "my-settings"]
}
# 3 创建数据流mymetrics-exam.prod，其创建方式与索引一样，只是多了一个@timestamp字段，并且能够匹配到索引模版，这时就会自动创建date stream 
POST mymetrics-exam.prod/_doc
{
  "@timestamp": "2021-01-01T00:00:00",
  "timestamp": "2021-01-01",
  "host": {
    "name": "xx"
  },
  "error": {
    "message": "xx"
  },
  "tags": [
    "ss",
    "ss"
  ]
}
# 查询数据流
GET mymetrics-exam.prod/_search
# 查询数据流分片分配情况
GET _cat/shards?v

# 9 题目给了一条数据，里面有个字段features，features是个数组，里面每个元素是{‘type’: ‘xxx’, ‘value’: ‘yyy’}，然后题目又给了一条查询语句并且告诉你改语句应该查不出任何结果，但是实际查出来了。要求创建一个新的索引features_reindex，reindex到新索引上，然后写一条查询语句，查不出结果。就是要把features换成nested类型，然后写一条查询语句来查询nested的内容。要求查询的数据{‘type’: ‘storage’, ‘vaue’: ’12’} 
GET features/_search
DELETE features_reindex
PUT features_reindex
{
  "mappings": {
    "properties": {
      "features": {
        "type": "nested"
      }
    }
  }
}
POST _reindex
{
  "source": {
    "index": "features"
  },
  "dest": {
    "index": "features_reindex"
  }
}
GET features_reindex/_search
GET features_reindex/_search
{
  "query": {
    "nested": {
      "path": "features",
      "query": {
        "bool": {
          "must": [
            {
              "match": {
                "features.type": "storage"
              }
            },
            {
              "match": {
                "features.value": "12"
              }
            }
          ]
        }
      }
    }
  }
}
# 10 给了两个索引，earthquakes_data和magnityde_type_desc。这两个索引里有一个共同的字段，并且是关联字段 magnitude_type，第二个索引里的第二个字段是desc，用来描述地震类型。要求创建一个新的索引earthquakes_data_new，里面包含了earthquakes_data里面的全部数据，并且每一条数据都要根据地震类型添加一个新的字段，就是desc。
DELETE earthquakes_data_new
GET earthquakes_data/_search
GET magnityde_type_desc/_search

PUT /_enrich/policy/my-policy
{
  "match": {
    "indices": "magnityde_type_desc",
    "match_field": "magnityde_type",
    "enrich_fields": ["desc"]
  }
}
PUT /_enrich/policy/my-policy/_execute
PUT _ingest/pipeline/task10_pipeline
{
  "processors": [
    {
      "enrich": {
        "policy_name": "my-policy",
        "field": "magnityde_type",
        "target_field": "new"
      }
    },
    {
      "script": {
        "lang": "painless",
        "source": """ 
            if(ctx.containsKey('new') && ctx.new.containsKey('desc')){
              ctx.desc = ctx.new.desc;
            }
            ctx.remove('new');
          """
      }
    }
  ]
}
DELETE earthquakes_data_new
POST _reindex
{
  "source": {
    "index": "earthquakes_data"
  },
  "dest": {
    "index": "earthquakes_data_new",
    "pipeline": "task10_pipeline"
  }
}
GET earthquakes_data_new/_search

```



# 第二套

0.环境准备

```
cluster1: node1,node2,node3,kibana1
cluster2: node4,kibana2
# 提前根据下列数据将索引数据插入对应集群中，没有说明的默认插入cluster1
不要在集群中添加remote_cluster角色
```



1.给出了一条数据，把这条数据插入数据流mylogds.prod，要求使得这条数据再开始5分钟在data_hot节点，然后立即滚动到data_warm节点，data_warm节点存在3分钟以后，移动到data_cold节点，在rollover发生6分钟后删除。匹配模板名称task1 匹配所有`mylogds*.*`

```
# 数据
{"name": "1"}
# 答案

```



2.将task2数据reindex到task2_new上，并且使得通过‘the’查询title，符合的条数为0

```
# 数据
PUT task2/_bulk
{"index":{}}
{"title":"This is the good one"}
{"index":{}}
{"title":"This is a good one"}
{"index":{}}
{"title":"look the flower"}
# 答案

```

3.索引task3中所有的doc都新增一个字段cross_address，由字段value01,value02,value03,value04拼接而成，中间用空格隔开

```
# 数据（原题value01-04均有值，这里增加了难度）
PUT task3/_bulk
{"index":{}}
{"value01":"www","value02":"baidu","value03":"com","value04":"cn"}
{"index":{}}
{"value01":"www","value02":"souhu","value03":"com"}
{"index":{}}
{"value02":"souhu","value03":"com","value04":"cn"}
{"index":{}}
{"value01":"www","value02":"souhu","value04":"cn"}
{"index":{}}
{"value01":"www","value02":"souhu","value04":null}
# 答案
```

4.创建角色task4_role,权限包括`create_snapshot`和`manage_iml`,该角色对所有索引由read和read_cross_cluster权限，创建用户task4_user，邮箱xxx@sina.com，密码password,匹配task4_role角色，并且该用户可登陆kibana操作

```
# 答案
```

5.task5索引查询时创建runtime字段profit，用revenue减去cost得到，并且按照小于0，0-1000，1000以上进行分组

```
# 数据
PUT task5/_bulk
{"index":{}}
{"revenue":1000,"cost":890}
{"index":{}}
{"revenue":1290,"cost":1000}
{"index":{}}
{"revenue":3245,"cost":2000}
{"index":{}}
{"revenue":700,"cost":800}
{"index":{}}
{"revenue":899,"cost":1000}
{"index":{}}
{"revenue":10000,"cost":100}
# 答案

```

6.查询task6索引中字段one,two,three中任意一个包含'fire'的数据，如果three包含则增加2.5倍评分，且评分规则为所有字段的评分和

```
# 数据
PUT task6/_bulk
{"index":{}}
{"one":"fire","two":"ok sad","three":"asds"}
{"index":{}}
{"one":"fire as","two":"asd","three":"fire"}
{"index":{}}
{"one":"123","two":"fire","three":"axda asd"}
{"index":{}}
{"one":"fire","two":"fire","three":"axda asd"}
{"index":{}}
{"one":"fire as","two":"xad","three":"axda asd"}
# 数据2
PUT task6/_bulk
{"index":{}}
{"one":"fire","two":"ok sad","three":"asds"}
{"index":{}}
{"one":"fireas","two":"asd","three":"fire"}
{"index":{}}
{"one":"123","two":"fire","three":"axda asd"}
{"index":{}}
{"one":"fire","two":"fire","three":"axda asd"}
{"index":{}}
{"one":"fireas","two":"xad","three":"axda asd"}
```

7.将kibana_sample_data_flights索引中的数据按照飞行距离DistanceKilometers进行分组，每1000米一组，计算每组的平均最小延迟时长FlightDelayMin，并计算最大的平均最小延迟时长

```
# 数据
来源于kibana的样例数据flight,请自行开启

```



8.从cluster2中复制索引task8到cluster1中，并重命名为task8_copy,并且查询task8_copy中price>50,tags包含'romantic comedy'的数据

```
# 数据，插入到cluster2中
PUT task8/_bulk
{"index":{"_id":1}}
{"title": "you are my love","tags":["romantic comedy","Action"],"price": 55}
{"index":{"_id":2}}
{"title": "you and me","tags":["Action","Tragedy"],"price": 30}
{"index":{"_id":3}}
{"title": "my life","tags":["Tragedy","Tragedy"],"price": 30}
{"index":{"_id":4}}
{"title": "We are always together","tags":["romantic comedy","Tragedy"],"price": 30}
{"index":{"_id":5}}
{"title": "me and my dog","tags":["romantic comedy","Action"],"price": 70}
{"index":{"_id":6}}
{"title": "me and my cat","tags":["comedy","Action"],"price": 90}
```



9.在cluster2中给task9创建快照,仓库路径/home/elastic/repo

```
# 数据 在cluster2中插入
PUT task9/_doc/1
{"name":1}
# 答案

```



10.创建一个搜索模版，用参数query_term查询字段one，并且将查询结果中字段one用标签`<strong></strong>`高亮显示，默认使用字段two逆序排序。并调用该模版，query_term参数设置为star

```
# 数据
PUT task10/_bulk
{"index":{}}
{"one":"star","two":"a"}
{"index":{}}
{"one":"star mark","two":"b"}
{"index":{}}
{"one":"star2 mark","two":"c"}
{"index":{}}
{"one":"all mark","two":"d"}
{"index":{}}
{"one":"all mark","two":"a"}
```

### 答案

```
# 1
PUT _cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.awareness.attributes": "hot_warm_cold" 
  }
}
# 创建ilm
PUT _ilm/policy/<policyName>
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_age": "5m",
            "max_primary_shard_size": "50gb"
          },
          "set_priority": {
            "priority": 100
          }
        },
        "min_age": "0ms"
      },
      "warm": {
        "min_age": "0m",
        "actions": {
          "set_priority": {
            "priority": 50
          },
          "allocate": {
            "require": {
              "hot_warm.cold": "data_warm"
            }
          }
        }
      },
      "cold": {
        "min_age": "3m",
        "actions": {
          "set_priority": {
            "priority": 0
          },
          "allocate": {
            "require": {
              "hot_warm.cold": "data_cold"
            }
          }
        }
      },
      "delete": {
        "min_age": "6m",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
# Creates a component template for index settings
PUT _component_template/my-settings
{
  "template": {
    "settings": {
      "index.lifecycle.name": "my_policy",
      "index.routing.allocation.require.hot_cold_warm": "data_hot",
      "number_of_shards": 1,
      "number_of_replicas": 0
    }
  }
}

PUT _index_template/task1
{
  "index_patterns": ["mylogds*.*"],
  "composed_of": [ "my-settings"],
  "data_stream": { }
}
 
DELETE _data_stream/mylogds.prod

# 错误答案
PUT mylogds.prod/_doc/1
{
  "message": "1",
  "@timestamp": "2021-01-01T00:00:00"
}
# 正确答案,数据流不能用PUT指定ID的形式添加
POST mylogds.prod/_doc
{
  "message": "1",
  "@timestamp": "2021-01-01T00:00:00"
}

这ILM理解有疑惑的可以参考这篇文章：https://articles.zsxq.com/id_e4qvdh99e8fa.html

#######
# 2
DELETE task2_new
PUT task2_new
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "english"
      }
    }
  }
}

POST _reindex
{
  "source": {
    "index": "task2"
  },
  "dest": {
    "index": "task2_new"
  }
}

GET task2_new/_search
{
  "query": {
    "match": {
      "title": "the"
    }
  }
}
#######
# 3
PUT _ingest/pipeline/task_pipeline
{
  "processors": [
    {
      "set": {
        "field": "cross_address",
        "value": "{{value01}} {{value02}} {{value03}} {{value04}}"
      }
    }
  ]
}

# 配置ingest角色
# 
POST task3/_update_by_query?pipeline=task_pipeline
GET task3/_search
# 4
# 先开启security
# 1生成证书
# 2
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate 
xpack.security.transport.ssl.client_authentication: required
xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: elastic-certificates.p12
# 生成密码
# 修改kibana配置
# 测试
GET task3/_search
PUT test/_doc/1
{"name":"q"}
 
# 5
GET task5/_search
GET task5/_search
{
  "size": 0, 
  "runtime_mappings": {
    "profit": {
      "type": "double",
      "script": {
        "source": "emit(doc['revenue'].value - doc['cost'].value)"
      }
    }
  },
  "aggs": {
    "profit_bucket": {
      "range": {
        "field": "profit",
        "ranges": [
          {
            "to": 0
          },
          {
            "from": 0,
            "to": 1000
          },
          {
            "from": 1000
          }
        ]
      }
    }
  }
}
# 6 
PUT task6/_bulk
{"index":{}}
{"one":"fire","two":"ok sad","three":"asds"}
{"index":{}}
{"one":"fire as","two":"asd","three":"fire"}
{"index":{}}
{"one":"123","two":"fire","three":"axda asd"}
{"index":{}}
{"one":"fire","two":"fire","three":"axda asd"}
{"index":{}}
{"one":"fire as","two":"xad","three":"axda asd"}

GET task6/_search
{
  "query": {
    "multi_match": {
      "query": "fire",
      "fields": ["one","two","three^2.5"],
      "type": "most_fields"
    }
  }
}
# 7
GET kibana_sample_data_flights/_mapping
GET kibana_sample_data_flights/_search

GET kibana_sample_data_flights/_search
{
  "size": 0, 
  "aggs": {
    "distance_bucket": {
      "histogram": {
        "field": "DistanceKilometers",
        "interval": 1000
      },
      "aggs": {
        "avg_delay": {
          "avg": {
            "field": "FlightDelayMin"
          }
        }
      }
    },
    "max_avg_delay": {
      "max_bucket": {
        "buckets_path": "distance_bucket>avg_delay"
      }
    }
  }
}
# 8
PUT _cluster/settings
{
  "persistent": {
    "cluster": {
      "remote": {
        "cluster_one": {
          "seeds": [
            "172.16.188.10:9300",
            "172.16.188.11:9300",
            "172.16.188.12:9300"
          ]
        },
        "cluster_two": {
          "seeds": [
            "172.16.188.8:9300"
          ]
        }
      }
    }
  }
}
PUT /task8_copy/_ccr/follow?wait_for_active_shards=1
{
  "remote_cluster" : "cluster_two",
  "leader_index" : "task8"
}

GET task8_copy/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "range": {
            "price": {
              "gte": 50
            }
          }
        },
        {
          "match_phrase": {
            "tags": "romantic comedy"
          }
        }
      ]
    }
  }
}
# 9 
PUT /_snapshot/my_backup
{
  "type": "fs",
  "settings": {
    "location": "/home/elastic/repo"
  }
}

PUT /_snapshot/my_backup/my_snapshot?wait_for_completion=true
{
  "indices": "task9",
  "ignore_unavailable": true,
  "include_global_state": false
}
DELETE task9
POST /_snapshot/my_backup/my_snapshot/_restore
{
  "indices": "task9",
  "ignore_unavailable": true,
  "include_global_state": false
}
GET task9/_search

# 10
GET task10/_search
PUT _scripts/my_search_tempalte
{
  "script": {
    "lang": "mustache",
    "source": {
      "query": {
        "term": {
          "one": {
            "value": "{{query_term}}"
          }
        }
      },
      "highlight": {
        "fields": {
          "one": {
            "pre_tags": [
              "<strong>"
            ],
            "post_tags": [
              "</strong>"
            ]
          }
        }
      },
      "sort": [
        {
          "two.keyword": {
            "order": "desc"
          }
        }
      ]
    }
  }
}


GET task10/_search/template
{
  "id": "my_search_tempalte", 
  "params": {
    "query_term": "star"
  }
}

```



## 拓展题

1.查询包含‘water’的数据，不区分大小写，应该输出4条

```
# 数据
PUT test/_bulk
{"index":{}}
{"name":"Watermelon"}
{"index":{}}
{"name":"Water"}
{"index":{}}
{"name":"wAter flow"}
{"index":{}}
{"name":"deepwaterflow"}
# 答案
GET test/_search
{
  "query": {
    "wildcard": {
      "name": {
        "value": "*water*",
        "case_insensitive": true
      }
    }
  }
}
```

### 





