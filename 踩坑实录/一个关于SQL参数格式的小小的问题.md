# 一个关于SQL参数格式的小小的问题

标签（空格分隔）： sql hql 参数

---
###问题来源
在“高能耗服务系统”的编写过程中，遇到过这样一个问题。
前段传过来中文参数值之后，想在后端编写hql查询语句，以完成“名称查询”这类的功能。
之前系统中的写法是
```java
List<FixInfo> fiList = fixInfoDao.find(
				"from FixInfo fi where fi.fiType=?", fiType);
```
这个写法实际上是把hql和参数分开给进去的，但是现在我有两个参数，就像直接写出hql字符串。问题就来了，格式一直存在问题，始终报错。
###解决办法
其实是查看了“mysql必知必会”才知道的。
之前我的写法都是这样的
```java
String hqlStr = "from FixInfo fi where fi.fiType= "+fiType+" and fi."+filtration+"= "+keyword;
```
这里两个参数，***filtration***与***fiType***是之前已经赋值的，所以我这里拼出的hql字符串是
```java
from FixInfo fi where fi.fiType= FixInfo and fi.fiCustomerName= 中国石油
```
刚开始报错我还以为是中文字符的问题。
后来看了书才发现:

* sql 里的参数是需要用 ' '（单引号） 来包起来的

所以其实hql的正确形式应该是：
```java
from FixInfo fi where fi.fiType= 'FixInfo' and fi.fiCustomerName= '中国石油'
```
###总的来说，就是一点:
* sql的相关参数一定要用 ' ' (单引号)括起来