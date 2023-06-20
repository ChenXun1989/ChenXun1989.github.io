---
title: solrcloud搭建
date: 2023-06-19 15:38:32
tags:
---
# solrcloud搭建
群里小伙伴需要一个solrcloud的解决案例。正好好久没碰过solr了。决定写个demo，顺便重新熟悉下solr。

1. solr版本：5.0
2. solr链接：http://archive.apache.org/dist/lucene/solr/5.0.0/
3. jdk版本：1.70+
4. windows环境安装：
> cd D:\app\solr-5.0.0\bin  
solr.cmd -c -z localhost:2181 -p 8983  
cd D:\app\solr-5.0.0-01\bin  
solr.cmd -c -z localhost:2181 -p 8984  
cd D:\app\solr-5.0.0-02\bin  
solr.cmd -c -z localhost:2181 -p 8985  
  
5. 注意端口号不要冲突-z 表示zookeeper连接配置 ，zookeeper服务要先启动。
   服务全部启动好，之后校验下是否成功。
> http://localhost:8983/solr/#/~cloud?view=tree

6. 创建conlection
> cd D:\app\solr-5.0.0\bin  
solr.cmd create_collection -c example -d ../example/example-DIH/solr/solr/conf/ -shards 3 -replicationFactor 2  

7. 参数说明：
  - -c : collection名称
  - -d : 配置文件的路径，可以使用上面提供的实例配置
  - -n : 配置名称可以和collection名称不同，默认这个参数不填的话，会使用collection名称作为config名称
  - -shards : 创建的shard个数，建议和集群节点数量一致。
  - -replicationFactor : 每个shard的副本数，综合考虑为了保证集群的稳定性，建议配置为 最少2个，最多集群节点数量/shard数量 * 2
校验结果：http://localhost:8983/solr/#/~cloud
服务端配置暂时告一段落，之后补上中文分词。
8. 客户端配置：
使用spring-boot-starter-solr来简化集成。核心是spring-data-solr

{% codeblock pom.xml lang:xml   %}

    <parent>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-parent</artifactId>  
    <version>1.4.0.RELEASE</version>  
    <relativePath/> <!-- lookup parent from repository -->  
    </parent>

    <properties>  
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>  
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>  
    <java.version>1.7</java.version>  
    </properties>  

    <dependencies>  
    <dependency>  
        <groupId>org.springframework.boot</groupId>  
        <artifactId>spring-boot-starter-data-solr</artifactId>  
    </dependency>  
    <dependency>  
        <groupId>org.projectlombok</groupId>  
        <artifactId>lombok</artifactId>  
    </dependency>  
    <dependency>  
        <groupId>org.springframework.boot</groupId>  
        <artifactId>spring-boot-starter-web</artifactId>  
    </dependency>  

    <dependency>  
        <groupId>org.springframework.boot</groupId>  
        <artifactId>spring-boot-starter-test</artifactId>  
        <scope>test</scope>  
    </dependency>  
    </dependencies>  

    <build>  
    <plugins>  
        <plugin>  
            <groupId>org.springframework.boot</groupId>  
            <artifactId>spring-boot-maven-plugin</artifactId>  
        </plugin>  
    </plugins>  
    </build>

{% endcodeblock %}

9. 剩下就是javaconfig了


{% codeblock  lang:java   %}

    /**
    * Project Name:chenxun-solr
    * File Name:SolrConfig.java
    * Package Name:com.chenxun.solr.config
    * Date:2016年8月20日下午3:11:32
    * Copyright (c) 2016, www midaigroup com Technology Co., Ltd. All Rights Reserved.
    * 
    */

    package com.chenxun.solr.config;
    
    import org.apache.solr.client.solrj.SolrClient;  
    import org.apache.solr.client.solrj.impl.CloudSolrClient;  
    import org.springframework.beans.factory.annotation.Value;  
    import org.springframework.context.annotation.Bean;  
    import org.springframework.context.annotation.Configuration;  
    import org.springframework.data.solr.core.SolrTemplate;  
    import org.springframework.data.solr.core.convert.CustomConversions;  
    import org.springframework.data.solr.core.convert.MappingSolrConverter;  
    import org.springframework.data.solr.core.convert.SolrConverter;  
    import org.springframework.data.solr.core.mapping.SimpleSolrMappingContext;  
    import org.springframework.data.solr.repository.config.EnableSolrRepositories;

    /**
      * ClassName:SolrConfig <br/>
      * Function: TODO ADD FUNCTION. <br/>
      * Reason: TODO ADD REASON. <br/>
      * Date: 2016年8月20日 下午3:11:32 <br/>
      * 
      * @author 陈勋
      * @version
      * @since JDK 1.7
      * @see
     */  
    @Configuration  
    @EnableSolrRepositories(basePackages = { "com.chenxun.solr" }, multicoreSupport = true)  
    public class SolrConfig {

    @Value("${spring.data.solr.zk-host}")  
    private String zkHost;

    @Bean  
    public SolrClient solrClient() {  
    return new CloudSolrClient(zkHost);  
    }

    @Bean  
    public SolrTemplate solrTemplate(SolrClient solrClient,SolrConverter solrConverter) throws Exception {  
    SolrTemplate solrTemplate =  new SolrTemplate(solrClient);  
    solrTemplate.setSolrConverter(solrConverter);  
    return solrTemplate;  
    }

    @Bean  
    public SolrConverter solrConverter(SimpleSolrMappingContext simpleSolrMappingContext,CustomConversions customConversions){  
    MappingSolrConverter solrConverter=new MappingSolrConverter(simpleSolrMappingContext);  
    solrConverter.setCustomConversions(customConversions);        
    return solrConverter;

    }

    @Bean  
    public SimpleSolrMappingContext simpleSolrMappingContext(){  
    SimpleSolrMappingContext simpleSolrMappingContext=new SimpleSolrMappingContext();  
    return simpleSolrMappingContext;  
    }

    @Bean  
    public CustomConversions customConversions(){  
    CustomConversions customConversions=new CustomConversions();  
    //  customConversions.registerConvertersIn(new GenericConversionService(){});  
    return customConversions;  
    }  
    }

{% endcodeblock %}
10. 实体


 {% codeblock  lang:java   %}

    /**
     * Project Name:chenxun-solr
      * File Name:Product.java
     * Package Name:com.chenxun.solr.model
     * Date:2016年8月20日下午4:48:36
     * Copyright (c) 2016, www midaigroup com Technology Co., Ltd. All Rights Reserved.
     * 
    */
    package com.chenxun.solr.model;

    import java.util.HashMap;  
    import java.util.List;  
    import java.util.Map;

    import lombok.Data;

    import org.apache.solr.client.solrj.beans.Field;  
    import org.springframework.data.annotation.Id;  
    import org.springframework.data.solr.core.mapping.Dynamic;  
    import org.springframework.data.solr.core.mapping.Indexed;  
    import org.springframework.data.solr.core.mapping.SolrDocument;  
    import org.springframework.stereotype.Component;

    /**
     * ClassName:Product <br/>
    * Function: TODO ADD FUNCTION. <br/>
    * Reason:   TODO ADD REASON. <br/>
    * Date:     2016年8月20日 下午4:48:36 <br/>
    * @author   陈勋
    * @version
    * @since    JDK 1.7
    * @see       
      */  
      @SolrDocument(solrCoreName="example") // Solr collection name  
      @Data  
      public class Product {

      @Field("id")                  // Specify field name in solr  
      @Id    
      private String id;

      @Field  
      private float price;

      @Field  
      private String name;

      @Field("*_s")  
      @Dynamic  
      private Map<String,String> map =new HashMap<String, String>();       
      }

{% endcodeblock %}


{% codeblock    lang:java   %}
    
    /** 
    Project Name:chenxun-solr
    * File Name:ProductRepository.java
    * Package Name:com.chenxun.solr.repository
    * Date:2016年8月20日下午4:54:24
    * Copyright (c) 2016, www midaigroup com Technology Co., Ltd. All Rights Reserved.
    * 
    */

    package com.chenxun.solr.repository;

    import org.springframework.data.solr.repository.SolrCrudRepository;

    import com.chenxun.solr.model.Product;

    /**
     * ClassName:ProductRepository <br/>
    * Function: TODO ADD FUNCTION. <br/>
    * Reason:   TODO ADD REASON. <br/>
    * Date:     2016年8月20日 下午4:54:24 <br/>
    * @author   陈勋
    * @version
    * @since    JDK 1.7
    * @see       
    */  
    public interface ProductRepository  extends SolrCrudRepository<Product, String>{       
    Iterable<Product>  findByName(String name);

}

{% endcodeblock %}

{% codeblock spring.properties   lang:xml   %}

    spring.data.solr.zk-host=localhost:2181

{% endcodeblock %}

{% codeblock  test.java  lang:java   %}


    package com.chenxun;

    import java.util.ArrayList;  
    import java.util.HashMap;  
    import java.util.Iterator;  
    import java.util.List;  
    import java.util.Map;

    import org.junit.Assert;  
    import org.junit.Test;  
    import org.junit.runner.RunWith;  
    import org.springframework.beans.factory.annotation.Autowired;  
    import org.springframework.boot.test.context.SpringBootTest;  
    import org.springframework.test.context.junit4.SpringRunner;

    import com.chenxun.solr.model.Product;  
    import com.chenxun.solr.repository.ProductRepository;

    @RunWith(SpringRunner.class)  
    @SpringBootTest  
    public class ChenxunSolrApplicationTests {

    @Autowired  
    private ProductRepository productRepository;  
  
    @Test  
    public void contextLoads() {  
        productRepository.deleteAll();  
        long count=productRepository.count();  
        Assert.assertTrue("empty count", count==0);  
        Product product=new Product();  
        product.setId("product-001");  
        product.setName("xxx");  
        Map<String,String> map=new HashMap<>();  
        map.put("key01", "abc");  
        map.put("key02", "123");  
        product.setMap(map);      
        productRepository.save(product);  
          
        Product product1=new Product();  
        product1.setId("product-002");    
        product1.setName("yyy");  
        Map<String,String> map1=new HashMap<>();  
        map1.put("key01", "abc1");  
        map1.put("key02", "1234");  
        product1.setMap(map1);        
        productRepository.save(product1);  
          
        count=productRepository.count();  
        Assert.assertTrue(count==2);  
          
          
        Iterator<Product> it=productRepository.findByName("yyy").iterator();  
        while(it.hasNext()){  
            Assert.assertTrue( it.next().getId().equals("product-002"));  
        }  
          
          
          
    }  

}



{% endcodeblock %}


到此大功告成，下一篇补上中文分词就OK了。
代码地址： https://github.com/ChenXun1989/chenxun-solr