## 通过QueryWrapper获取sql片段
```java
//获取targetSql ex: (org_code like ?)
String targetSql = queryWrapper.getTargetSql();
//把条件从QueryWrapper中取出来 {nwvalue1:xxx} 这里只有getTargetSql后才会有值
Map<String, Object> paramNameValuePairs = queryWrapper.getParamNameValuePairs();
Collection<Object> values = paramNameValuePairs.values();
for (Object value : values) {
    //替换?为 'value'
    targetSql = targetSql.replace("?", "'" + value + "'");
}
```
使用时使用 `apply(targetSql) `即可
