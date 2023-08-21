# js页面通过url传值到后端，中文乱码的解决

标签（空格分隔）： js 前端 乱码

---
###问题来源
在“高能耗服务系统”的编写过程中，存在这样一个场景。
在js函数中通过异步的方式从后台取得一系列值，然后处理，最后发送到一个弹出式的jsp中，在这个jsp中使用echarts来进行数据的可视化展示。
其中参数的传递采用的是url传值的方法，但是传递中文的值的时候，遇到了乱码的问题，这样的问题使得jsp中的字符串处理遇到了问题，进而导致echarts框架不能正常显示。

###解决办法
上网查找，找到了相关的方法。
附上链接：[http://www.cnblogs.com/nicholas_f/archive/2009/03/27/1423263.html]

在传递参数前将中文参数进行两次编码，jsp页面获取参数后对中文参数进行一次解码，中文参数就不会变为乱码了！

参考例子：
```java
<%@ page language="java" contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8"%>
<%@ page import="java.net.*" %>
<%
String str0="";
String str1="";
      if(request.getParameter("param0")!=null){
        str0=request.getParameter("param0");//直接获取中文参数
       }
try{
     if(request.getParameter("param1")!=null){
       str1=URLDecoder.decode(request.getParameter("param1"),"utf-8");//对中文参数进行解码
      }
}catch(Exception e){
     e.printStackTrace();
  }
%> 
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
<script type="text/javascript">
var str="你好";
function test0(){
window.location="Test.jsp?param0="+str;//直接传递中文参数
}
function test1(){
window.location="Test.jsp?param1="+encodeURI(encodeURI(str));//对中文参数进行双层编码后再传递
}
</script>
</head>
<body>
<input value=<%=str0 %>>
<input type="button" value="乱码" onclick="test0()"><br>
<input value=<%=str1 %>>
<input type="button" value="正常" onclick="test1()">
</body>
</html>
```

###总的来说，就是两点:

* 在js中两次使用"encodeURI()"
* 然后再java代码中使用"java.net.URLDecoder.decode()"

happy~