## ES常见数据类型及其区别

## ES有哪几种查询

## ES默认评分机制：BM25
    TF/IDF基础上进一步区分不同字段长度对分数的贡献，更长的字段出现关键字的贡献低于更短的字段。

## 如何自定义ES评分

## ES排序操作注意事项

## 常见分词器。如何将中文转换为拼音？
- 中文分词器IK_MAX_WORD
  特点：尽可能将文本切分出所有可能的词语，生成最细粒度的分词结果。
适用场景：适用于搜索需要高召回率的场景。
优点：分词粒度细，召回率高。
缺点：分词结果数量多，可能会增加索引的存储空间和查询时的计算开销。
示例：
假设文本是 "中华人民共和国国歌"

ik_max_word 分词结果：["中华", "华人", "人民共和国", "人民", "共和", "国歌", "国"]
它会将文本中所有可能的词语都切分出来。

- 中文分词器IK_SMART
  特点：按最粗粒度对文本进行分词，生成较少的分词结果。
适用场景：适用于对性能要求较高、只关心核心词语的场景。
优点：分词数量少，索引空间更小，查询性能更好。
缺点：召回率较低，分词可能遗漏部分词语。
示例：
同样的文本 "中华人民共和国国歌"

ik_smart 分词结果：["中华人民共和国", "国歌"]
它只会提取出核心词语，忽略细粒度的词。
- 

## ES控制评分机制
1. boost boost 的作用是对查询或某个字段的得分进行加权，常见的表现方式是将 boost 值直接乘以原始得分。举个例子：如果一个查询的得分是 5，并且你为这个查询设置了 boost=2，那么该查询的最终得分将变成 5 * 2 = 10。
2. 


## 实战：如何实现同时根据中文、拼音和拼音首字母进行搜索
1. 思路概述
使用 拼音分词器 将中文内容转换成 完整拼音 和 首字母。
使用 标准中文分词器 保留原始的中文分词。
在索引配置中使用多字段（multi-fields）或自定义分词器，分别处理中文、拼音和拼音首字母搜索。
2. 具体实现步骤
a. 安装拼音分词器插件
安装 Elasticsearch 的 拼音分词器插件：

```bash
bin/elasticsearch-plugin install analysis-pinyin
```

安装完成后重启 Elasticsearch 服务。
b. 配置索引
创建索引，配置自定义分析器来处理多种搜索场景：

```json
PUT chat_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "pinyin_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "pinyin_filter"]
        },
        "pinyin_first_letter_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "pinyin_first_letter_filter"]
        },
        "ik_max_word_analyzer": {
          "type": "ik_max_word"    // 中文分词器
        }
      },
      "filter": {
        "pinyin_filter": {
          "type": "pinyin",
          "keep_full_pinyin": true,           // 保留完整拼音
          "keep_original": false,
          "lowercase": true
        },
        "pinyin_first_letter_filter": {
          "type": "pinyin",
          "keep_first_letter": true,         // 保留拼音首字母
          "keep_original": false,
          "lowercase": true
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "message": {
        "type": "text",
        "fields": {
          "chinese": {                       // 中文分词
            "type": "text",
            "analyzer": "ik_max_word_analyzer"
          },
          "pinyin": {                        // 完整拼音分词
            "type": "text",
            "analyzer": "pinyin_analyzer"
          },
          "pinyin_first": {                  // 拼音首字母分词
            "type": "text",
            "analyzer": "pinyin_first_letter_analyzer"
          }
        }
      }
    }
  }
}
```

配置说明：
message.chinese：用于存储中文分词后的数据，支持直接输入中文进行搜索。
message.pinyin：使用拼音分词器，生成完整的拼音，支持输入拼音进行搜索。
message.pinyin_first：生成拼音首字母，支持输入拼音首字母进行搜索。
c. 索引数据
假设存储的聊天内容是 "张三" 和 "李四"：

```json
POST chat_index/_doc/1
{
  "message": "张三"
}

POST chat_index/_doc/2
{
  "message": "李四"
}
```

d. 查询数据

```json
复制代码
POST chat_index/_search
{
  "query": {
    "multi_match": {
      "query": "zs",
      "fields": ["message.chinese", "message.pinyin", "message.pinyin_first", "message"]
    }
  }
}
```

## 实战：优先返回活跃用户
```json
{
  "query": {
    "function_score": {
      "query": {
        "match_all": {}
      },
      "functions": [
        {
          "gauss": {
            "last_login_time": {
              "origin": "now",
              "scale": "10h",
              "offset": "5m",
              "decay": 0.7
            }
          }
        }
      ],
      "boost_mode": "sum",
      "score_mode": "sum"
    }
  }
}
```
参数解释
1. gauss： 衰减函数使用高斯函数。高斯函数在起始阶段衰减较慢而后衰减速度逐渐增大。
1. origin
origin 表示高斯函数的起始点，通常设置为当前时间（now）。它定义了参考点，从这个参考点开始计算衰减。

在上述示例中，origin: "now" 表示参考点是当前时刻。如果 last_login_time 离当前时刻较近，那么它的分数就会较高。

2. scale
scale 控制曲线的宽度，即影响衰减的速度。它指定了从 origin 到某个距离后，文档分数会达到一半衰减的距离。

例如，scale: "1m" 表示在 1 分钟内，分数会衰减一半。更大的 scale 值会使分数衰减得更加平缓，而较小的 scale 值则使得分数衰减得更快。
3. offset
offset 设置了距离 origin 的一个“偏移量”，它决定了在多远的距离后开始进行衰减。它主要用于调整衰减的起始点。

例如，offset: "10s" 表示在当前时间后 10 秒钟内的用户不会受到衰减，衰减效果会在 10 秒后开始生效。如果没有设置 offset，则衰减会立即开始。
4. decay
decay 控制衰减的速率，影响高斯曲线的斜率。较大的 decay 值会使得分数衰减得更快，较小的 decay 值则会让分数衰减得更慢。

例如，decay: 0.5 会导致分数衰减得相对较慢，decay: 1 则表示标准的高斯衰减，分数衰减较快。



## 实战：分数计算

```json
{
  "query": {
    "bool": {
      "should": [
        {
          "bool":{
            "should":[
              {"match":{"des":"Elasticsearch"}},
              {"match":{"tag":"Elasticsearch"}}
            ]
          }
        },
        {"match":{"info":"Elasticsearch"}},
        {
          "function_score": {
            "query": { "match": { "title": "Elasticsearch" } },
            "functions": [
              {
                "gauss": {
                  "created_at": {
                    "origin": "2024-01-01",
                    "scale": "30d",
                    "decay": 0.5
                  }
                }
              },
              {
                "field_value_factor": {
                  "field": "popularity",
                  "factor": 1.5,
                  "missing": 1
                }
              }
            ],
            "score_mode": "sum",
            "boost_mode": "multiply"
          }
        },
        {
          "function_score": {
            "query": { "term": { "category": "search" } },
            "functions": [
              {
                "linear": {
                  "updated_at": {
                    "origin": "2024-01-10",
                    "scale": "10d",
                    "decay": 0.8
                  }
                }
              }
            ],
            "score_mode": "multiply",
            "boost_mode": "sum"
          }
        }
      ]
    }
  }
}

```
查询结构拆解
1. 顶层 bool 查询：
should 子句包含 4 个子查询。
bool 查询的 should 子句 会将所有匹配条件的得分相加（默认情况下）。
2. 四个 should 子句的分数：
(1) 子查询 1：嵌套的 bool 查询
json
复制代码
{
  "bool": {
    "should": [
      {"match": {"des": "Elasticsearch"}},
      {"match": {"tag": "Elasticsearch"}}
    ]
  }
}
这里的 should 查询包含两个 match 查询。
这两个 match 查询分别对 des 和 tag 字段执行匹配。
两者的分数会根据 match 查询 的得分规则计算，且由于处于 bool 查询的 should 中，它们的分数会相加。
(2) 子查询 2：简单 match 查询
json
复制代码
{"match": {"info": "Elasticsearch"}}
这个查询对 info 字段执行 match 查询。
它的分数直接由 match 查询的相关性评分决定。
(3) 子查询 3：第一个 function_score 查询
json
复制代码
{
  "function_score": {
    "query": { "match": { "title": "Elasticsearch" } },
    "functions": [
      {
        "gauss": {
          "created_at": {
            "origin": "2024-01-01",
            "scale": "30d",
            "decay": 0.5
          }
        }
      },
      {
        "field_value_factor": {
          "field": "popularity",
          "factor": 1.5,
          "missing": 1
        }
      }
    ],
    "score_mode": "sum",
    "boost_mode": "multiply"
  }
}
query 部分：对 title 字段执行 match 查询，计算出初始分数。
functions 部分：
gauss 衰减函数根据 created_at 字段的时间距离（高斯函数），生成一个分数衰减值。
field_value_factor 根据 popularity 字段值，按因子 1.5 进行分数调整。
score_mode: sum：
两个 functions（gauss 和 field_value_factor）的分数会被相加。
boost_mode: multiply：
query 得分（match 计算的分数）会与 functions 计算得到的分数之和 相乘，形成最终得分。
(4) 子查询 4：第二个 function_score 查询
json
复制代码
{
  "function_score": {
    "query": { "term": { "category": "search" } },
    "functions": [
      {
        "linear": {
          "updated_at": {
            "origin": "2024-01-10",
            "scale": "10d",
            "decay": 0.8
          }
        }
      }
    ],
    "score_mode": "multiply",
    "boost_mode": "sum"
  }
}
query 部分：对 category 执行 term 查询。term 查询的初始分数是 1.0。
functions 部分：
linear 衰减函数根据 updated_at 字段的时间距离，生成一个分数衰减值。
score_mode: multiply：
由于只有一个函数，这里的分数就是 linear 衰减函数的输出值。
boost_mode: sum：
query 的分数（1.0）会与 functions 计算的分数相加，形成最终得分。
最终分数合并规则
1. 每个 should 子句会计算出自己的分数。
2. function_score 查询内部的分数由 score_mode 和 boost_mode 控制。
3. 顶层 bool 查询的 should 子句 会将 4 个子查询的分数相加，形成最终的综合分数。
分数合并总结
第一个 should 子句：

由 des 和 tag 字段的 match 查询分数相加。
第二个 should 子句：

info 字段的 match 查询分数。
第三个 should 子句：

title 字段的 match 分数 乘以 functions 部分的分数之和（由 score_mode: sum 计算）。
第四个 should 子句：

term 查询的分数（1.0）加上 functions 部分的分数（由 score_mode: multiply 计算）。
最终，顶层 bool 查询将所有 should 子句的分数相加，作为最终的文档得分。



