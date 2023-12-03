# SQL 快速撸码方法总结

[TOC]

## 先导
sql 代码的

## 标注

### 查询标记

宗旨：标记的标定对象是数据集、及各个数据集嵌套关系

 标记编号时心中默念： **" XXXX数据由N个部分组成，第M个部分是XXXX "**

```
-- 1.5.3.4.2 'girl' monster.woman 少女与老虎
-- 一眼看出子查询层级，与层级顺序，相当于多叉树的节点序号串
-- 一眼看出子查询结果集的别名
-- 一眼看出数据来源（如果子查询是叶子结点的话）
```
此标记处的代码，属于：
第 **1** 个 select 
所需要的第 **5** 个子查询中
所需要的第 **3** 个子查询中
所需要的第 **4** 个子查询中
所需要的第 **2** 个子查询
他的别名被叫做 **‘girl’**，
其数据来自于 **‘monster’** 库的 **‘woman’** 表，

对于这条标记的说明是 **‘少女与老虎‘**。

>补充说明：多个子查询进行 union并为组成的合集命名时，名字只标在 第一个子查询上就可以了

### 连接标记

当连接与子查询相邻时，可以省略第二个连接参数 b（好像多数情况都可以省略）
暂定：key 取 **“依附方”** 的连接字段 
| 连接类型 | 标记符号 |
|:-:|:-:|
| [] | 之前的连接结果 |
| left join | a <|key| b |
| full join | a <==> b |
| inner join | a >|key|< b |
| union | a + b |
| union all | a ++ b |


## 格式化示例与标记的应用
宗旨：既然逻辑上是一颗树，那就尽量真的写成树的形状

> 格式说明：
>
> 1、同一级的括号，必须tab到同一列，（case/end处除外）
>
> 2、带名字标记标在名字对应的括号首次出现处
>
> 3、不带名字的标记，标在 select 处
>
> 4、and 退后一格显示其 “从属” 身份
>
> 5、同级 select、from、where 对齐
>
> 6、from后子查询退后一格
>
> 7、子查询 select相较于其前方的“（” 退后一格
>
> 8、同级别语句，如“join”、"and“ 后面的括号保持同列即可（7.8可保证标记深度与代码缩进深度一致）

```
select   													-- 1
	id,
	a.first
from
	(   													-- 1.1 'aa' 有括号的记得要深一级
		select      											-- 1.1.1 db.table1
			id,
			min(
				case 
					when number in ('1', '2', '3', '11') 
						and age<18 then number
                	when name = 'dad' then 'dad'
					else null 
				end) num,	--num数值
    	from
			db.table1
    	where
			age <= '18'
    	group by
			number

    	union all                    							-- [] <=>
		select                                                	-- 1.1.2 没括号基本不会出错
			passenger_id,
    	from
        	(       											-- 1.1.2.1 'a' 
				select												-- 1.1.2.1.1 db3.tab3
					passenger_id,
        		from
					db3.tab3
        		where	age <= '20'
        		and
				(
					product_id in (1, 2, 3, 4, 6, 7, 9, 20, 11, 19, 12, 21, 22, 26)
                	or product_id is null
            	)
        		group by
					passenger_id,
			) a
    	group by
			passenger_id
	) aa

	LEFT OUTER JOIN            									-- [] <|first|second|
	(   														-- 1.2 'bb' 
		select														-- 1.2.1 db2.tab2
			passenger_id,
		from
			db2.tab2
		where
			concat(year, month, day) = '20180417'
	) bb
	ON
	(
    	aa.first = bb.firsrt
    	and aa.second = bb.second
	)

GROUP BY
	aa.first
	aa.second
;
```