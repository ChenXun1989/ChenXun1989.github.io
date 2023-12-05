---
title: 基于jquery把表单转成成json对象
tags:
  - 前端
abbrlink: 11819
date: 2023-06-20 13:38:15
---
# 基于jquery把表单转成成json对象
最近前端框架修改，小伙伴希望能像以前写jsp一样使用 对象.属性作为表单元素的name值
{% codeblock  lang:javascript   %}

     <form id="test-form">   
       <input name="a.b.c" value="abc">  
       <input name="a.b.d" value="abd">  
       <input name="x" value="x">  
       <input name="a.f" value="af">  
       <input name="y" value="0">  
       <input name="y" value="y2">  
       <input name="m[0].c" value="m0c">  
       <input name="m[0].d" value="m0d">  
       <input name="m[1].c" value="m1c">  
       <input name="m[1].d" value="m1d">   
      <input type="button" id="test-btn">  
    </form>  

{% endcodeblock %}

我这种懒人第一反应肯定去找度娘，发现度娘居然没有，只好自己造了轮子。

{% codeblock  lang:javascript   %}

    function form2Json(params){  
    var selector=params.form;  
    var values= $(selector).serializeArray();  
    var obj={};  
    for (var index = 0; index < values.length; ++index)      
    {      
    var temp=obj; //上一级       
    var n=values[index].name;    
    if(n.indexOf(".")>-1){    
    var arr=n.split(".");    
    for(var i=0;i<arr.length-1;i++){       
    if(arr[i].indexOf("[")>-1){    
    var a=arr[i].substring(0,arr[i].indexOf("["));    
    temp[a]=temp[a]||[];    
    var y=arr[i].substring(arr[i].indexOf("[")+1,arr[i].indexOf("]"));    
    temp[a][y]=temp[a][y]||{};    
    temp=temp[a][y];    
    }else{    
    temp[arr[i]]=temp[arr[i]] || {};       
    temp=temp[arr[i]];    
    }    
    }  
    temp[arr[arr.length-1]]=values[index].value;         
    }else{    
    if(obj[n] !==undefined && obj[n]!=null){    
    if( !$.isArray(obj[n])){    
    var v=obj[n];    
    obj[n]=[];    
    obj[n].push(v);    
    }
    
                obj[n].push(values[index].value);    
             }else{     
                 obj[n]=values[index].value;    
             }    
         }       
    }       
    return obj;


{% endcodeblock %}

该轮子支持复杂对象，支持对象数组，支持简单属性
{% codeblock  lang:javascript   %}

    $("#test-btn").on("click",function(){  
    var json=form2Json({  
    form:"#test-form"  
    });  
    console.log(json);  
    console.log(JSON.stringify(json));  
    });

{% endcodeblock %}

最终结果如下

{% codeblock  lang:javascript   %}

    {"a":{"b":{"c":"abc","d":"abd"},"f":"af"},"x":"x","y":["0","y2"],"m":[{"c":"m0c","d":"m0d"},{"c":"m1c","d":"m1d"}]}

{% endcodeblock %}