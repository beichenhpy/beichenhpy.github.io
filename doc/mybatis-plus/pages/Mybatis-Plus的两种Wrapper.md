# 一、Wrapper种类
可分为以下两种

- 普通Wrapper 如QueryWrapper
- lambdaWrapper 如 LambdaQueryWrapper
## 1、顶层AbstractWrapper
继承于`AbstractWrapper`
这里可以看到，MP巧用泛型，将Column设置为泛型R 这样就可以用户自定义输入的column
```java
public abstract class AbstractWrapper<T, R, Children extends AbstractWrapper<T, R, Children>> extends Wrapper<T>
    implements Compare<Children, R>, Nested<Children, Children>, Join<Children>, Func<Children, R>{
    //例如 
     @Override
    public Children eq(boolean condition, R column, Object val) {
        return addCondition(condition, column, EQ, val);
    }
    //可被重写
     protected String columnToString(R column) {
        return (String) column;
    }
    }
```
## 2、QueryWrapper
`QueryWrapper`
```java
public class QueryWrapper<T> extends AbstractWrapper<T, String, QueryWrapper<T>>
    implements Query<QueryWrapper<T>, T, String>
```
## 3、LambdaQueryWrapper
`LambdaQueryWrapper` 是`AbstractLambdaWrapper`的子类
```java
public class LambdaQueryWrapper<T> extends AbstractLambdaWrapper<T, LambdaQueryWrapper<T>>
    implements Query<LambdaQueryWrapper<T>, T, SFunction<T, ?>>
```
在`AbstractLambdaWrapper`中，实现了
```java
public abstract class AbstractLambdaWrapper<T, Children extends AbstractLambdaWrapper<T, Children>>
    extends AbstractWrapper<T, SFunction<T, ?>, Children>{
    // 此方法在父类 AbstractWrapper 中是强转为String
     @Override
    protected String columnToString(SFunction<T, ?> column) {
        return columnToString(column, true);
    }
   }
```
# 二、推荐用法
推荐使用 `Wrappers` 工具类创建 Wrapper
# 三、执行过程
以LambdaQueryWrapper为例
```java
LambdaQueryWrapper<User> queryWrapper = Wrappers.lambdaQuery(User.class);
queryWrapper.eq(User::getName(),"梨花");
```
上面新建了一个wrapper调用了eq方法
这里的eq将会流向他的父类--->`AbstractLambdaWrapper` 然后在父类中并没有找到此方法
所以流向父类--->`AbstractWrapper<T, SFunction<T, ?>, Children>`
注意此处的泛型 `R` 已经被替换为了 ` SFunction<T, ?>`
因此将会对`AbstractWrapper`中的所有`R`的泛型赋予类型，现在查看顶层类中的 `eq方法`
```java
 	@Override
    public Children eq(boolean condition, R column, Object val) {
        return addCondition(condition, column, EQ, val);
    }


    /**
     * 普通查询条件
     *
     * @param condition  是否执行
     * @param column     属性
     * @param sqlKeyword SQL 关键词
     * @param val        条件值
     */
    protected Children addCondition(boolean condition, R column, SqlKeyword sqlKeyword, Object val) {
        return doIt(condition, () -> columnToString(column), sqlKeyword, () -> formatSql("{0}", val));
    }


    /**
     * 对sql片段进行组装
     *
     * @param condition   是否执行
     * @param sqlSegments sql片段数组
     * @return children
     */
    protected Children doIt(boolean condition, ISqlSegment... sqlSegments) {
        if (condition) {
            expression.add(sqlSegments);
        }
        return typedThis;
    }

```
上面可以看到执行了 `addCondition`方法，在方法中执行了 `doIt` 方法
在执行中，使用lambda去返回一个查询字段 `sqlSegments`
这里主要关注 `columnToString(column)`方法，当子类未重写此方法时将会执行以下默认方法
```java

    /**
     * 获取 columnName
     */
    protected String columnToString(R column) {
        return (String) column;
    }
```
但是子类 `AbstractLambdaWrapper`中实现了此方法。
但是不要忘了此使执行这个方法的对象还是最底层的子类：`LambdaQueryWrapper` ；
根据Java类调用原理：【子类可以调用父辈类的protected/public的方法，并且方法调用会先从子类重写的方法开始找】
会优先使用该类或者该类的父辈类重写的方法，如果没有重写方法，就默认使用顶层类的方法。
所以会从 `LambdaQueryWrapper` 开始寻找，如果没有则去父类`AbstractLambdaWrapper`中寻找 `columnToString`方法找到了就执行，【否则就先当于本类的内部方法调用】。
具体代码见 文章 《**为了将某实体类的成员变量换为下划线》最后的代码**
