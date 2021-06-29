## 关键字介绍
反序列化：字节流变成实体 即前端传值后端  
序列化：实体变成字节流 即后端返回值给前端
## 前情提要
心血来潮想了解下这两个注解在序列化和反序列化时，到底哪个生效
### 测试@JsonFormat
关于网上说的`@JsonFormat`只能格式化当实体类序列化时的Date类型->是不正确的
经过我的测试
代码如下
```java
/**
* pattern = "yyyy-MM-dd HH:mm:ss"
* 测试用例：2021-03-10T11:30:07.365+00:00 不能反序列化 可以将Date序列化成 pattern格式的字符串
* 测试用例 2021-03-11 20:20:10 可以反序列化和序列化
* pattern = "yyyy/MM/dd"
* 测试用例 2021/03/10 可以反序列化和序列化
* 测试用例 2021/03/10T11:30:07.365+00:00 可以反序列化和序列化
* 可见 只要用例满足pattern的格式要求，就可以进行反序列化和序列化
*/
@JsonFormat(timezone = "GMT+8",pattern = "yyyy-MM-dd HH:mm:ss")
private Date birthday;
```
### 测试@DateTimeFormat
只有传入以下四种类型的值，才可以被反序列化

- "yyyy-MM-dd'T'HH:mm:ss.SSSX",
- "yyyy-MM-dd'T'HH:mm:ss.SSS",
- "EEE, dd MMM yyyy HH:mm:ss zzz",
- "yyyy-MM-dd"

代码如下
```java
/**
 * pattern = "yyyy-MM-dd HH:mm:ss" 时
 *测试用例 2021-03-10T11:30:07.365+00:00 可以反序列化
 *测试用例 2021-03-11 20:20:10 可以反序列化
 * pattern = "yyyy/MM/dd"时任何值都不能反序列化
 */
@DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss")
private Date birthday;

```
### 小结
| 注解 | 使用范围 |
| --- | --- |
| @JsonFormat | 序列化和反序列化均可，需要满足pattern的类型才能转换 |
| @DateTimeFormat | 只有反序列化生效将String转为Date，传值需要满足类型
- yyyy-MM-dd'T'HH:mm:ss.SSSX",
- "yyyy-MM-dd'T'HH:mm:ss.SSS",
- "EEE, dd MMM yyyy HH:mm:ss zzz",
- "yyyy-MM-dd"
 |

# 总结
`@DateTimeFormat`很鸡肋 只能格式化特殊的四种日期格式
大部分的需求都可以使用 `@JsonFormat`解决
只有当传值为 
```java
@JsonFormat(timezone = "GMT+8",pattern = "yyyy-MM-dd HH:mm:ss")
//@JsonFormat(timezone = "GMT+8",pattern = "HH:mm:ss")
//@JsonFormat(timezone = "GMT+8",pattern = "yyyy-MM-dd")
//@JsonFormat(timezone = "GMT+8",pattern = "yyyy/MM/dd")
private Date birthday;
//使用以上的注解，传入 2021-03-10T11:30:07.365+00:00 格式或者
// 2021/03/10T11:30:07.365+00:00也可以进行解析
```
如下代码 2021-03-10T11:30:07.365+00:00 这种类型，并且需要 格式化为 yyyy-MM-dd HH:mm:ss这种时
需要结合`@DateTimeFormat` 一起使用
```java
@JsonFormat(timezone = "GMT+8",pattern = "yyyy-MM-dd HH:mm:ss")
@DateTimeFormat("yyyy-MM-dd HH:mm:ss")
private Date birthday;
```
## 如果不想记太多，那么就一起用吧，不会有什么问题的
