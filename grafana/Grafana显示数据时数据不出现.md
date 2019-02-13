# Grafana显示数据时数据不出现

时间： 20180710

## 原因：grafana的sql不是原生sql

```sql
SELECT
  UNIX_TIMESTAMP(time) as 'time_sec',
  price as 'person_count'
FROM test1
WHERE $__timeFilter(time)
group by  time
ORDER BY time ASC
```



这里面有些东西是不能够省略的。比如UNIX_TIMESTAMP、'time_sec'、$__timeFilter等关键字。



这里面每条记录都需要有一个timestamp类型的字段。在这个test1表中的字段就是time，计数用的字段即price。