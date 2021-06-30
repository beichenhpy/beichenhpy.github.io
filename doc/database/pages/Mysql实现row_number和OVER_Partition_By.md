# 0x00 初始化表

```sql
CREATE TABLE `test` (
  `month` varchar(100) DEFAULT NULL,
  `count` int(11) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
INSERT INTO lgaic_yf_test.test (`month`,count) VALUES
	 ('202001',2),
	 ('202001',3),
	 ('202001',1),
	 ('202001',1),
	 ('202002',5),
	 ('202002',1),
	 ('202002',1);
```

# 0x01 实现row_number()

具体的思路就是，初始化变量为0，一次递加  

```sql
-- row_number()
select x.*,(@r:=@r+1) as rank  from (
select *
from test t 
-- 这里规定好排序顺序
order by t.`month` asc,t.count desc) x,
-- 初始化行数 = 0
(select @r:=0) r ;
```
# 0x02 OVER(Partition By field) 
`field` ----->代表某个字段，需要按照某一个字段进行分组
```sql
-- row_number() + 分组
-- part为month 对应用谁分组
-- 注意 order by的顺序
select z.month,z.count,z.rank
from 
(select
	x.*,
	-- 第一行为 1
	@rownum := @rownum + 1,
	-- 判断要分组的字段 如果是当前字段则 rank + 1 否则 重置为1
	if(@part = x.month,@r := @r + 1,@r := 1) as rank,
	-- 更新@part
	@part := x.month
from
	(
	-- 这部分比较重要，由于没有order by 所以需要在这里将要分组的数据进行排序好
	select
		*
	from
		test e
	order by
		e.month asc,e.count desc) x,
	(
	select
	-- 初始化 rownum = 0,part分组信息 = null ,rank = 0
		@rownum := 0,
		@part := null,
		@r := 0) rt
		)z
```

## 拓展
根据上面的按month分组后，就可以取到，每个month对应的count前几名

```sql
select *
from z
-- 前两名
where z.rank in (1,2)
```

# 0x03 引用
[MySQL如何实现row_number()及row_number over(partition by column)](https://blog.csdn.net/zhouseawater/article/details/90697305)
[postgresql OVER() Partition By Order By](https://blog.csdn.net/u010773514/article/details/102581055)
