## 实现查询某组数据中最新的几条数据
```sql
select * from post inner join 
(
  select user_id, max(create_time) as max_time from post group by user_id
) tmp
on tmp.max_time = post.create_time;
```
## 实现查询某组数据中user_id出现频率大于3的数据
```sql
select * from post inner join
(
  select count(user_id) as c,user_id from post group by user_id having count(user_id) > 3
) tmp
on post.user_id = tmp.user_id; 
```
