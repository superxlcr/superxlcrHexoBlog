---
title: lucene(Kibana)搜索规则
tags: [杂项]
categories: [杂项]
date: 2017-12-29 15:38:40
description: （转载）语法关键字、查询词(Term)、查询域(Field)、通配符查询(Wildcard)、模糊查询(Fuzzy)、临近查询(Proximity)、区间查询(Range)、增加一个查询词的权重(Boost)、布尔操作符、组合
---

本文转载自：http://blog.csdn.net/lwwgtm/article/details/60353812

kibanna用的是lucene的搜索规则

Lucene所支持的查询语法可见http://lucene.apache.org/java/3_0_1/queryparsersyntax.html

# 语法关键字


+ - && || ! ( ) { } [ ] ^ " ~ x ? : \

如果所要查询的查询词中本身包含关键字，则需要用\进行转义

# 查询词(Term)


Lucene支持两种查询词，一种是单一查询词，如"hello"，一种是词组(phrase)，如"hello world"。

# 查询域(Field)


在查询语句中，可以指定从哪个域中寻找查询词，如果不指定，则从默认域中查找。

查询域和查询词之间用:分隔，如title:"Do it right"。

:仅对紧跟其后的查询词起作用，如果title:Do it right，则仅表示在title中查询Do，而it right要在默认域中查询。

# 通配符查询(Wildcard)


支持两种通配符：?表示一个字符，x表示多个字符。

通配符可以出现在查询词的中间或者末尾，如te?t，testx，text，但决不能出现在开始，如xtest，?test。

# 模糊查询(Fuzzy)


模糊查询的算法是基于Levenshtein Distance，也即当两个词的差别小于某个比例的时候，就算匹配，如roam~0.8，即表示差别小于0.2，相似度大于0.8才算匹配。

# 临近查询(Proximity)


在词组后面跟随~10，表示词组中的多个词之间的距离之和不超过10，则满足查询。

所谓词之间的距离，即查询词组中词为满足和目标词组相同的最小移动次数。

如索引中有词组"apple boy cat"。

如果查询词为"apple boy cat"~0，则匹配。

如果查询词为"boy apple cat"~2，距离设为2方能匹配，设为1则不能匹配。

| (0) | boy | apple | cat | 
| - | - | - | - |
| (1) |   | boyapple | cat | 
| (2) | apple | boy | cat | 

如果查询词为"cat boy apple"~4，距离设为4方能匹配。

| (0) | cat | boy | apple | 
| - | - | - | - |
| (1) |   | catboy | apple | 
| (2) |   | boy | catapple | 
| (3) |   | boyapple | cat | 
| (4) | apple | boy | cat | 

 

# 区间查询(Range)


区间查询包含两种，一种是包含边界，用[A TO B]指定，一种是不包含边界，用{A TO B}指定。

如date:[20020101 TO 20030101]，当然区间查询不仅仅用于时间，如title:{Aida TO Carmen}

# 增加一个查询词的权重(Boost)


可以在查询词后面加^N来设定此查询词的权重，默认是1，如果N大于1，则说明此查询词更重要，如果N小于1，则说明此查询词更不重要。

如jakarta^4 apache，"jakarta apache"^4 "Apache Lucene"

# 布尔操作符


布尔操作符包括连接符，如AND，OR，和修饰符，如NOT，+，-。

默认状态下，空格被认为是OR的关系，QueryParser.setDefaultOperator(Operator.AND)设置为空格为AND。

+表示一个查询语句是必须满足的(required)，NOT和-表示一个查询语句是不能满足的(prohibited)。

# 组合


可以用括号，将查询语句进行组合，从而设定优先级。

如(jakarta OR apache) AND website

 

Lucene的查询语法是由QueryParser来进行解析，从而生成查询对象的。

通过编译原理我们知道，解析一个语法表达式，需要经过词法分析和语法分析的过程，也即需要词法分析器和语法分析器。

QueryParser是通过JavaCC来生成词法分析器和语法分析器的。
