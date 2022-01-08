# 测试 添加索引
post /_bulk
{
"index": {
"_index": "product",
"_id": "1"
}
}
{
"name": "小米手机",
"desc": "手机中的战斗机"
}

#给product index增加别名
post /_aliases
{
"actions": [
{
"add": {
"index": "product",
"alias": "product_template"
}
}
]
}

GET _cluster/allocation/explain

GET _cat/health?v

GET kibana_sample_data_flights/_search

GET kibana_sample_data_flights

get _cat/indices

GET product/_mapping

DELETE produt2

PUT product2/_doc/1
{
"title":"i like u"
}

get product2/_mapping

put product3
{
"mappings": {
"properties" : {
"title" : {
"type" : "text",
"analyzer": "english",
"fields" : {
"keyword" : {
"type" : "keyword",
"ignore_above" : 6
}
}
}
}
}
}

GET product3/_mapping
GET product3/_search

GET product3/_search
{
"query":{
"term": {
"title.keyword": {
"value": "i lik"
}
}
}
}

PUT product3/_doc/1
{
"title":"i like u"
}

########### nested
DELETE order
PUT /order/_doc/1
{
"order_name": "xiaomi order",
"desc": "shouji zhong de zhandouji",
"goods_count": 3,
"total_price": 12699,
"goods_list": [
{
"name": "xiaomi PRO MAX 5G",
"price": 1999
},
{
"name": "ganghuamo",
"price": 19
},
{
"name": "shoujike",
"price": 1999
}
]
}
PUT /order/_doc/2
{
"order_name": "Cleaning robot order",
"desc": "shouji zhong de zhandouji",
"goods_count": 2,
"total_price": 12699,
"goods_list": [
{
"name": "xiaomi cleaning robot order",
"price": 1999
},
{
"name": "dishwasher",
"price": 4999
}
]
}

PUT order
{
"mappings": {

         "properties" : {
        "desc" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
        "goods_count" : {
          "type" : "long"
        },
        "goods_list" : {
          "type": "nested", 
          "properties" : {
            "name" : {
              "type" : "text",
              "fields" : {
                "keyword" : {
                  "type" : "keyword",
                  "ignore_above" : 256
                }
              }
            },
            "price" : {
              "type" : "long"
            }
          }
        },
        "order_name" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
        "total_price" : {
          "type" : "long"
        }
      }
}
}

GET order/_mapping
GET order/_search

GET order/_search
{
"_source": false,
"query": {
"bool": {
"must": [
{
"match": {
"goods_list.name": "dishwasher"
}
},
{
"match": {
"goods_list.price": 4999
}
}
]
}
}
}


GET order/_search
{
"query": {
"nested": {
"path": "goods_list",
"query": {
"bool": {
"must": [
{
"match": {
"goods_list.name": "dishwasher"
}
},
{
"match": {
"goods_list.price": 4999
}
}
]
}
}
}
}
}


GET product/_search
{
"query": {
"match_all": {}
}
}

#term 搜索方式 对搜索词不分词
#keyword 数据类型 对原数据字段不分词

GET product/_search
{
"query": {
"term": {
"name.keyword": {
"value": "xiaomi phone"
}
}
}
}

GET product/_search
{
"query": {
"match": {
"name.keyword": "xiaomi phone"
}
}
}

GET product/_analyze
{
"analyzer": "standard",
"text": ["xiaomi phone"]
}


#Template
#Dynamtic Template
PUT my-index-000001
{
"mappings": {
"dynamic_templates": [
{
"integers": {
"match_mapping_type": "long",
"mapping": {
"type": "integer"
}
}
},
{
"my_string": {
"match_mapping_type": "string",
"mapping": {
"type": "text",
"fields": {
"raw": {
"type": "keyword",
"ignore_above": 256
}
}
}
}
}
]
}
}

POST my-index-000001/_doc
{
"age": 12323,
"long_sdsffef": 34342,
"assdder_text": "asdsadsdds"
}

GET my-index-000001/_mapping

# reindex

DELETE my_index
GET my_index/_mapping
POST _reindex
{
"source": {
"index": "order"
},
"dest": {
"index": "order_new"
}
}


PUT order_new
{
"mappings": {

}
}
}



GET product/_search
{
"from": 0,
"size": 20,
"query": {
"match_all": {}
}
, "sort": [
{
"price": {
"order": "asc"
}
},
{
"_score":{
"order": "desc"
}
}
]
}


# term match_phrase keyword
# term 在搜索的时候不会江搜索词进行分词
# keyword 是字段类型，是指原数据不进行分词
# match_phrase 分词结果必须在被检索的字段中包含，而且顺序必须一致，默认必须是连续的

GET product/_search



GET product/_search
{
"query": {
"match_phrase": {
"name": "xiaomi nfc"
}
}
}

GET product/_search
{
"query": {
"term": {
"name.keyword": {
"value": "xiaomi nfc phone"
}
}
}
}
GET product/_search
{
"query": {
"term": {
"name": {
"value": "xiaomi nfc phone"
}
}
}
}

GET product/_search
{
"query": {
"match": {
"name.keyword": "xiaomi nfc phone"
}
}
}

GET product/_search
{
"query": {
"range": {
"price": {
"gte": 999,
"lte": 2999
}
}
}
}

# 组合查询 bool query
GET product/_search
{
"query": {
"bool": {
"must": [
{
"match": {
"name": "xiaomi"
}
},
{
"term": {
"price": {
"value": "4999"
}
}
}
]
}
}
}



