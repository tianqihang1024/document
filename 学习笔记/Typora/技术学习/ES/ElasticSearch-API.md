





## @Document

标识要持久化到 `Elasticsearch` 的域对象，可以根据自身需求定义内部参数。

- indexName：索引的名称（一般与实体对应即可）
- shards：索引 indexName() 的分片数。用于创建索引。在 4.0 版中，默认值从 5 更改为 1，以反映 Elasticsearch 的默认设置的更改，在 Elasticsearch 7.0 中也更改为 1。
- replicas：索引 indexName() 的副本数。用于创建索引。



## @Field

标识要持久化到 `Elasticsearch` 域对象的属性，可以根据自身需求定义内部参数。

- type：标识持久化到`Elasticsearch`后的属性类型，取值范围参考`FieldType`。为空的话会自动识别（通常没有问题，但是一些对`Elasticsearch`来说的特殊属性，持久化后的效果可能出乎你的意料）
- analyzer：创建索引时按照定义的analyzer对属性内容进行分词，有定义用定义，没有的话使用ES预设的
- searchAnalyzer：查询时按照定义的searchanalyzer对查询的内容进行分词，没有定义使用analyzer，或者ES预设的



## FieldType

@Field注解中type属性的enum类，包含了所有elasticsearch的数据类型，下面介绍常用的几种。

- Text：字符串文本，创建索引时会自动分词
- Keyword：关键词，创建索引时不会进行分词
- Date：date格式可以在创建索引时指定format参数，不指定的话，会使用默认格式：`strict_date_optional_time||epoch_millis`。（其他文章上说的`GTC+8`问题，持久化类型选择时间戳时暂未遇到）
    1. 在elasticsearch中存储数据用的是json，而json没有date这个属性，因此date只能是int、long的时间戳或者字符串。
    2. strict_date_optional_time：年份、月份、天必须分别以4位、2位、2位表示，不满足放不进`es`中（具体请上网查找）
    3. `epoch_millis`：约束值必须大于等于`Long.MIN_VALUE`，小于等于`Long.MAX_VALUE`
    4. `format`：查看`DateFormat`枚举类
- Long：同Java
- Integer：同Java