## 分析器analyzer主要构成
Elasticsearch 分析器主要包括 三个核心组件：

| 组件                      | 作用                                                                                                         | 示例                           |
| :------------------------ | :----------------------------------------------------------------------------------------------------------- | :----------------------------- |
| 字符过滤器 (Char Filter)  | 对输入的文本进行预处理，比如去掉 HTML 标签、替换特定字符等。多个字符过滤器可以同时使用，按声明顺序依次执行。 | HTML Strip、Mapping            |
| 分词器 (Tokenizer)        | 将输入的文本根据规则拆分成一个个词元（Token），是分析器的核心部分。                                          | Standard Tokenizer、Whitespace |
| 词元过滤器 (Token Filter) | 对分词器生成的词元进一步处理，比如小写化、去除停用词、同义词替换、词干提取等。                               | Lowercase、Stopword、Synonym   |


## 常用内置组件介绍
### 字符过滤器 (Char Filter)
1. html_strip ： 移除 HTML 或 XML 标签，保留文本内容。
2. mapping ： 将特定字符或字符序列替换为其他字符或序列。不过需要提供一个映射文件或映射列表。
  示例：
  
  ```json
    PUT /example_index
    {
    "settings": {
        "analysis": {
        "char_filter": {
            "my_mapping": {
            "type": "mapping",
            "mappings": [
                "cat => dog",
                "rat => mouse"
            ]
            }
        },
        "analyzer": {
            "custom_mapping_analyzer": {
            "type": "custom",
            "char_filter": ["my_mapping"],
            "tokenizer": "standard"
            }
        }
        }
    }
    }
  ```
3. pattern_replace ：基于正则表达式匹配字符，并用替换字符串替换匹配到的字符。
  配置参数：
    pattern：正则表达式。
    replacement：替换的字符串
    ```json
        PUT /example_index
        {
        "settings": {
            "analysis": {
            "char_filter": {
                "my_pattern_replace": {
                "type": "pattern_replace",
                "pattern": "dog",
                "replacement": "cat"
                }
            },
            "analyzer": {
                "custom_pattern_analyzer": {
                "type": "custom",
                "char_filter": ["my_pattern_replace"],
                "tokenizer": "standard"
                }
            }
            }
        }
        }
    ```
### 分词器（Tokenizer）
1. Standard Tokenizer
默认分词器，基于 Unicode 文本分割标准。
按照单词进行分割，去除标点符号和空白字符。
会将大写字母转换为小写（标准化）。
适用场景：英文、简单西方语言。
2. Whitespace Tokenizer
根据空格分割文本。
不进行任何大小写转换或标点符号处理。
示例：
输入：Hello World!
输出：Hello、World!
3. Keyword Tokenizer
不分割文本，将整个输入当作一个单独的 token。
示例：
输入：Hello World!
输出：Hello World!
4. Letter Tokenizer
将非字母字符作为分隔符进行分割。
示例：
输入：Hello123World
输出：Hello、World
5. Lowercase Tokenizer
和 Letter Tokenizer 类似，但会将所有字符转换为小写。
示例：
输入：Hello123WORLD
输出：hello、world
6. Pattern Tokenizer
使用 正则表达式 进行分割。
默认正则：\W+，即非单词字符（如空格、标点等）。
示例：
输入：Hello-World_123，正则：[^\w]+
输出：Hello、World、123
可配置参数：
pattern：自定义正则表达式。
7. NGram Tokenizer
将输入文本拆分为 固定长度的子字符串（N-Grams）。
配置参数：
min_gram：最小子串长度（默认 1）。
max_gram：最大子串长度（默认 2）。
示例：
输入：hello，min_gram=2，max_gram=3
输出：he、hel、el、ell、ll、llo
8. Edge NGram Tokenizer
从文本 开头 开始拆分成 N-Gram 子字符串，适合前缀匹配。
配置参数：
min_gram：最小子串长度。
max_gram：最大子串长度。
示例：
输入：hello，min_gram=2，max_gram=3
输出：he、hel
9. UAX-29 URL Email Tokenizer
基于 Unicode Text Segmentation 标准（UAX #29）。
专门处理 URL 和 Email。
示例：
输入：user@example.com
输出：user@example.com
10. Path Hierarchy Tokenizer
适用于路径分割，主要用于文件系统路径或分层数据。
示例：
输入：/a/b/c
输出：/a、/a/b、/a/b/c
可配置参数：
delimiter：路径分隔符（默认 /）。
replacement：替换分隔符（可选）。
skip：跳过的层级（默认 0）。
11. Char Group Tokenizer
按照指定的字符分组分割文本。
示例：
配置：tokenize_on_chars: ["whitespace", "punctuation"]
输入：Hello, world!
输出：Hello、world
12. Classic Tokenizer
适用于旧版本 Lucene 分词器，主要针对英文文本。
和 Standard Tokenizer 类似，但处理更加简单，移除了一些复杂的 Unicode 处理。
13. Simple Pattern Tokenizer
使用 简单正则表达式 进行文本分割。
与 Pattern Tokenizer 不同，使用更简单的正则表达式引擎。
示例：
正则：\\d+（匹配数字）
输入：token123token456
输出：token、token




## 词元过滤器（Token Filter）

| 过滤器          | 功能                          | 示例                                      |
| :-------------- | :---------------------------- | :---------------------------------------- |
| Lowercase       | 转为小写                      | HELLO -> hello                            |
| Stop            | 移除停用词                    | the quick fox -> quick fox                |
| Stemmer         | 词干化                        | running -> run                            |
| Synonym         | 同义词替换                    | quick -> fast                             |
| Edge NGram      | 生成前缀 N-Grams              | hello -> he, hel, hell, hello             |
| NGram           | 生成 N-Grams                  | hello -> he, hel, el, ll, lo              |
| Shingle         | 生成 Shingles（多个词项组合） | quick brown fox -> quick brown, brown fox |
| Length          | 按长度过滤词元                | the fox -> fox                            |
| Pinyin          | 汉字转换为拼音                | 张三 -> zhang, san                        |
| Pattern Replace | 使用正则表达式替换词元        | hello 123 -> hello NUM                    |
| Decimal Digit   | 处理数字字符                  | 100 -> 100                                |
| Unique          | 移除重复词元                  | hello hello world -> hello world          |


## 常见的分析器
Elasticsearch 内置了多种常见的分析器，以下是一些常见分析器及其特点：

| 分析器            | 组成结构                                        | 作用                                                              |
| :---------------- | :---------------------------------------------- | :---------------------------------------------------------------- |
| Standard 分析器   | Standard Tokenizer + Lowercase Filter           | 默认分析器，按语言学规则分词，小写化所有词元。                    |
| Whitespace 分析器 | Whitespace Tokenizer                            | 按空格拆分文本，不进行其他处理。                                  |
| Simple 分析器     | Letter Tokenizer + Lowercase Filter             | 仅按字母分词，所有词元小写化，非字母字符视为分隔符。              |
| Stop 分析器       | Standard Tokenizer + Lowercase + Stopword       | 在 Standard 分析器的基础上，删除常见停用词（如 "the"、"is" 等）。 |
| Keyword 分析器    | Keyword Tokenizer                               | 不分词，保留整个输入文本作为一个词元。                            |
| Pattern 分析器    | Pattern Tokenizer + 可选的 Filters              | 使用正则表达式进行分词。                                          |
| Language 分析器   | 不同语言的分析器（如 English、French 等）       | 提供语言特定的分词和过滤，如英文词干提取、停用词去除等。          |
| Custom 分析器     | 自定义组合 Char Filter、Tokenizer、Token Filter | 用户可以组合不同的组件，自定义分析器满足特定需求。                |

## 实站1.自定义分析器
可以通过自定义分析器来组合字符过滤器、分词器和词元过滤器。例如：

自定义中文拼音分析器

```json
PUT /my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "custom_pinyin_analyzer": {
          "type": "custom",
          "tokenizer": "standard",                      // 标准分词器
          "filter": [
            "lowercase",                               // 转小写
            "pinyin",                                  // 自定义拼音过滤器
            "stop"                                     // 去除停用词
          ]
        }
      },
      "filter": {
        "pinyin": {
          "type": "pinyin",
          "keep_first_letter": true,                   // 保留拼音首字母
          "keep_separate_first_letter": false,
          "limit_first_letter_length": 16,
          "lowercase": true
        }
      }
    }
  }
}
应用该分析器
json
复制代码
POST /my_index/_analyze
{
  "analyzer": "custom_pinyin_analyzer",
  "text": "张三"
}
输出结果：
[zhang, z, san, s]

## 实战2.通过文件配置停用词
在文件系统中创建一个文本文件，如 stopwords.txt，并写入停用词列表，每个停用词占一行。

txt
复制代码
the
is
in
at
a
an
配置分析器使用该文件中的停用词。例子如下：

json
复制代码
"settings": {
  "analysis": {
    "tokenizer": {
      "type": "standard"
    },
    "filter": {
      "stop_filter": {
        "type": "stop",
        "stopwords_path": "path_to_your_stopwords_file/stopwords.txt"
      }
    },
    "analyzer": {
      "custom_stop_analyzer": {
        "type": "custom",
        "tokenizer": "standard",
        "filter": ["stop_filter"]
      }
    }
  }
}
注意：
stopwords_path 指向的路径是相对于 Elasticsearch 配置文件中的 config 目录的路径，或者是绝对路径。
你可以使用内置的停用词列表，如 "stopwords": "_english_"，来加载英文的默认停用词。

## 实战3.配置同义词
你需要创建一个包含同义词的文本文件（如 synonyms.txt），文件内容格式如下：

txt
复制代码
# 用逗号分隔的多个单词直接互为同义词
quick, fast
# 箭头代表单向同义词，如果左边的同义词是右边，但是右边的同义词不是左边
jump => leap
smart, intelligent
然后在 synonym_filter 中使用 synonyms_path 指定该文件的路径：

json
复制代码
"settings": {
  "analysis": {
    "tokenizer": {
      "type": "standard"
    },
    "filter": {
      "synonym_filter": {
        "type": "synonym",
        "synonyms_path": "path_to_your_synonym_file/synonyms.txt"
      }
    },
    "analyzer": {
      "custom_synonym_analyzer": {
        "type": "custom",
        "tokenizer": "standard",
        "filter": ["synonym_filter"]
      }
    }
  }
}
注意：
synonyms_path 指向的路径也是相对于 Elasticsearch 配置文件中的 config 目录的路径，或者是绝对路径。
在文件中，同义词对应该是用逗号分隔的，每个同义词组放在一行。
你可以使用 双向同义词：如果在同义词文件中有 quick, fast，表示 quick 和 fast 互为同义词。
