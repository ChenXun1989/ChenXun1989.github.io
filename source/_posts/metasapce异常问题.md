---
title: metasapce异常问题
tags:
  - jvm
  - 生产问题实战
abbrlink: 6806f9db
date: 2025-04-19 16:16:48
---
# 问题背景
小伙伴反馈服务越来越卡，查看arms监控发现，服务jvm元空间一直在涨，正常情况下，这个应该是不动的，或者微弱波动。
{% asset_img 1.png  image %}
# 分析思路和过程
- 技术直觉 元空间一直涨，一般情况下怀疑对象有 ，动态class，groovy脚本之类
- 小伙伴代码排查了下没有显示使用 动态class，groovy脚本之类 ，不排除三方jar里面使用
- jmap -histo <pid>   生成快照 ,full gc前
- {% asset_img 2.png  image %}
- class对象过多，有24w之多，重大嫌疑对象是这个 fastjson的 ASMClassLoader
- jmap -dump:format=b,file=/tmp/heapdump.hprof  <pid>   生成 dump文件 （这个一样注意不要加 live 参数）
- visualVm 分析dump文件
- {% asset_img 4.png  image %}
- 查了下fastjson资料，SerializeConfig 会通过 ASMClassLoader 生成动态class，推荐姿势是复用
- 对象引用发现可疑业务代码 ApiParameter.class

# 问题代码
{% codeblock WebApiSdkUtil.java  lang:java   %}

    public class WebApiSdkUtil<T> {
    private static final Logger log = LoggerFactory.getLogger(WebApiSdkUtil.class);
    private SortedMap<String, Object> parameters;
    private SerializeConfig config = new SerializeConfig();
    private String app_id;
    private String app_key;
    private ConnectConfig connectConfig;
    
    
    public T postJsonEx(String url, TypeReference<T> type) throws Exception {
    try {
    this.initSign();
    ApiParameter apiParameter = new ApiParameter();
    apiParameter.clearParameters();
    apiParameter.setParams(this.parameters);
    apiParameter.setConnectConfig(this.connectConfig);
    String response = HttpClientUtil.doPostJsonMap(apiParameter, url);
    return (T)JSON.parseObject(response, type, new Feature[0]);
    } catch (Exception e) {
    throw e;
    }
    
    }

{% endcodeblock %}

{% codeblock ApiParameter.java  lang:java   %}

    public ApiParameter() {
    this.config = new SerializeConfig();
    this.appId = "xxxx";
    this.appKey = "xxxx";
    this.parameters = new TreeMap();
    this.parameters.put(SpecialParameter.OPERATION.getCode(), OperationEnum.SELECT.getCode());
    this.parameters.put(SpecialParameter.IS_KEEP_NULL.getCode(), YesOrNo.NO.getCode());
    this.config.propertyNamingStrategy = PropertyNamingStrategy.SnakeCase;
    }

{% endcodeblock %}

> WebApiSdkUtil类，执行postJsonEx方法，会生成ApiParameter对象，这个对象会 new SerializeConfig对象，导致元空间一直涨，
一直到classloader卸载才能回收

# 后续思考
发现问题，到定位原因，以及修复花了些许时间，在Ai时代，此类问题是否在Ai的支持下，能否做到自动发现，自动修复，耗时缩短到几分钟呢。

