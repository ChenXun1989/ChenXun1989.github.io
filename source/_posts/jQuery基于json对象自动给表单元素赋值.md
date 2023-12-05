---
title: jQuery基于json对象自动给表单元素赋值
tags:
  - 前端
abbrlink: 58164
date: 2023-06-20 12:30:39
---
# jQuery基于json对象自动给表单元素赋值

为了提高前端小伙伴的开发效率，造了个基于json对象根据表单元素的name属性自动赋值的轮子

{% codeblock  lang:javascript   %}

    json2form:function(obj){  
    var nodeParent = null;  
    var value = undefined;  
    var $el = null;  
    var nodeName = "";  
    for(var i in obj){  
    value= obj[i] ;  
    if(value === undefined || value===null){  
    continue;  
    }  
    if(typeof value == 'object'){  
    nodeParent=obj.nodeParent;  
    value.nodeParent=nodeParent?nodeParent+"."+i : i;  
    if(value instanceof Array){  
    for(var mm=0;mm<value.length;mm++){  
    var ms=value[mm];  
    if(typeof ms == 'object'){  
    nodeParent=ms.nodeParent;  
    ms.nodeParent=ms.nodeParent?ms.nodeParent+"."+i+"["+mm+"]":i+"["+mm+"]";  
    arguments.callee(ms);  
    }  
    }  
    $el=$("[name='"+i+"']");  
    if($el.is(":checkbox")){  
    $el.each(function(){  
    if($(this).val() == value){  
    $(this).prop("checked",true);  
    }  
    })  
    }  
    else if($el.is(":radio")){  
    $el.each(function(){  
    if($(this).val() == value){  
    $(this).prop("checked",true);  
    }  
    })  
    }  
    }else{  //递归  
    arguments.callee(value);  
    }  
    }  
    else{  
    nodeName=obj.nodeParent?obj.nodeParent+"."+i : i ;  
    $el=$("[name='"+nodeName+"']");  
    if($el.length > 0){  
    // console.log("匹配数据名称："+nodeName+"值："+value);  
    if($el.is(":text")||$el.attr("type")=="hidden"){  
    if($el.data("money") && $el.data("money") == "money"){  
    value = outputdollars(value);  
    }  
    $el.val(value);

                 }else if($el.is(":radio")){  
                     $el.each(function(){  
                        if($(this).val()==value){  
                            $(this).prop("checked",true);  
                        }  
                     })  
                 }  
                 else if($el.is("select")){  
                    $el.find("option").filter(function(){return $(this).val() == obj[i];}).prop("selected",true);  
                 }else if($el.is("textarea")){  
                    $el.val(value)  
                 }  
              }  
          }  
      }      

    }

{% endcodeblock %}

注意： 表单的name属于与json对象的属性名为一致，保持继承链。
例如 input name=’a.b.c’ 表示json对象里面的a属性里面的b属性的c属性。

