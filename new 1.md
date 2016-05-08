来写一发关于hibernate的XML映射文档的故事

先来看一下代码

<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE hibernate-mapping PUBLIC "-//Hibernate/Hibernate Mapping DTD 3.0//EN"
"http://www.hibernate.org/dtd/hibernate-mapping-3.0.dtd">


<hibernate-mapping>
    <class name="com.pojo.Userinfo" table="userinfo" catalog="baidudemo">
        <id name="id" type="java.lang.Integer">
            <column name="id" />
            <generator class="native" />
        </id>
        <property name="name" type="java.lang.String">
            <column name="name" not-null="true" />
        </property>
        <property name="password" type="java.lang.String">
            <column name="password" not-null="true" />
        </property>
    </class>
</hibernate-mapping>


首次来看ID的属性

<id name="id" type="java.lang.Integer">
		<column name="id" />
		<generator class="native" />
</id>
id标签用来将实体类中的特定属性映射成标签。
name（可选）嘛，当然就是属性名字啦，
type(可选)是字段的类型，
column（可选）是数据表中主键的字段名字，默认值是name的属性值
<generator></generator>这个标签有说道了，是主键的生成策略，默认是assigned，意思是主键值由程序为其赋值


映射多对一的关系
