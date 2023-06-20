---
title: solr中文分词
date: 2023-06-19 15:29:14
tags:
- solr
---
# solr中文分词

solr中文分词。
solr5.0 自带一个中文分词包，lucene-analyzers-smartcn-5.0.0.jar。
在安装目录下搜寻找到，并copy到solr提供的web服务目录的lib目录下。
修改collection配置里面的schema.xml。新增字段类型。

{% codeblock schema.xml lang:xml   %}
<fieldType name="text_cn" class="solr.TextField" positionIncrementGap="100">    
<analyzer type="index">    
<!-- 此处需要配置主要的分词类 -->    
<tokenizer class="solr.SmartChineseSentenceTokenizerFactory"></tokenizer>    
<filter class="solr.SmartChineseWordTokenFilterFactory"/>    
</analyzer>    
<analyzer type="query">    
<!-- 此处配置同上 -->    
<tokenizer class="solr.SmartChineseSentenceTokenizerFactory"/>    
<filter class="solr.SmartChineseWordTokenFilterFactory"/>    
</analyzer>    
</fieldType>  
{% endcodeblock %}

启动solr cloud服务，新建collection就好。

