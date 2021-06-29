### 一、缓存TableInfo
1、首先可以知道的是会初始化表，缓存在TableInfo这个实体类中
大概知道是使用静态方法（`TableInfoHelper`->`initTableFields`）
2、这个方法有一段是用来寻找主键的
```java
// 标记是否读取到主键
        boolean isReadPK = false;
        // 是否存在 @TableId 注解
        boolean existTableId = isExistTableId(list);
        // 是否存在 @TableLogic 注解
        boolean existTableLogic = isExistTableLogic(list);

        List<TableFieldInfo> fieldList = new ArrayList<>(list.size());
        for (Field field : list) {
            if (excludeProperty.contains(field.getName())) {
                continue;
            }

            /* 主键ID 初始化 */
            if (existTableId) {
                TableId tableId = field.getAnnotation(TableId.class);
                if (tableId != null) {
                    if (isReadPK) {
                        throw ExceptionUtils.mpe("@TableId can't more than one in Class: \"%s\".", clazz.getName());
                    } else {
                        initTableIdWithAnnotation(dbConfig, tableInfo, field, tableId, reflector);
                        isReadPK = true;
                        continue;
                    }
                }
            } else if (!isReadPK) {
                isReadPK = initTableIdWithoutAnnotation(dbConfig, tableInfo, field, reflector);
                if (isReadPK) {
                    continue;
                }
            }
```
3、关注 initTableIdWithAnnotation和initTableIdWithoutAnnotation方法
这两个分别是为有注解 `@TableId` 和 没有注解的用来初始化主键的
主要关注这个赋值 设置Key主键相关信息
```java
tableInfo.setKeyRelated(checkRelated(underCamel, property, column))
            .setKeyColumn(column)
            .setKeyProperty(property)
            .setKeyType(keyType);
```
### 二、运行期间进行的触发设置Key
1、在MybatisParameterHandler中
```java
protected void populateKeys(TableInfo tableInfo, MetaObject metaObject, Object entity) {
        final IdType idType = tableInfo.getIdType();
        final String keyProperty = tableInfo.getKeyProperty();
        if (StringUtils.isNotBlank(keyProperty) && null != idType && idType.getKey() >= 3) {
            // 如果用户设置了ID值 则使用用户的
            final IdentifierGenerator identifierGenerator = GlobalConfigUtils.getGlobalConfig(this.configuration).getIdentifierGenerator();
            Object idValue = metaObject.getValue(keyProperty);
            // 检查Id值是否为null
            if (StringUtils.checkValNull(idValue)) {
                //判断ID类型是否为雪花算法
                if (idType.getKey() == IdType.ASSIGN_ID.getKey()) {
                    if (Number.class.isAssignableFrom(tableInfo.getKeyType())) {
                        //如果是Long类型 直接赋值Long
                        metaObject.setValue(keyProperty, identifierGenerator.nextId(entity));
                    } else {
                        //否则转换为String
                        metaObject.setValue(keyProperty, identifierGenerator.nextId(entity).toString());
                    }
                } else if (idType.getKey() == IdType.ASSIGN_UUID.getKey()) {
                    metaObject.setValue(keyProperty, identifierGenerator.nextUUID(entity));
                }
            }
        }
    }
```
### 三、关注雪花算法的实现
`identifierGenerator.nextId(entity)`
作者为了拓展性，将 IdentifierGenerator 设置为了 interface
```java
public interface IdentifierGenerator {

    /**
     * 生成Id
     *
     * @param entity 实体
     * @return id
     */
    Number nextId(Object entity);

    /**
     * 生成uuid
     *
     * @param entity 实体
     * @return uuid
     */
    default String nextUUID(Object entity) {
        return IdWorker.get32UUID();
    }
}
```
在`MybatisParameterHandler`中，使用了全局设置中配置好的，默认使用 `DefaultIdentifierGenerator`
```java
final IdentifierGenerator identifierGenerator = GlobalConfigUtils.getGlobalConfig(this.configuration).getIdentifierGenerator();
```
介绍 `DefaultIdentifierGenerator `
```java
public class DefaultIdentifierGenerator implements IdentifierGenerator {
	// 雪花算法
    private final Sequence sequence;
	
    //无参构造方法，直接内部初始化雪花算法，使用雪花算法的构造函数
    public DefaultIdentifierGenerator() {
        this.sequence = new Sequence();
    }
	// 可以在初始化时设置对应的 工作id和数据id
    public DefaultIdentifierGenerator(long workerId, long dataCenterId) {
        this.sequence = new Sequence(workerId, dataCenterId);
    }
	
    //直接初始化指定的算法
    public DefaultIdentifierGenerator(Sequence sequence) {
        this.sequence = sequence;
    }
	//获取id
    @Override
    public Long nextId(Object entity) {
        return sequence.nextId();
    }
}
```
介绍 `Sequence` 为雪花算法的主要实现类
```java
public class Sequence {

    private static final Log logger = LogFactory.getLog(Sequence.class);
    /**
     * 时间起始标记点，作为基准，一般取系统的最近时间（一旦确定不能变动）
     */
    private final long twepoch = 1288834974657L;
    /**
     * 机器标识位数
     */
    private final long workerIdBits = 5L;
    private final long datacenterIdBits = 5L;
    private final long maxWorkerId = -1L ^ (-1L << workerIdBits);
    private final long maxDatacenterId = -1L ^ (-1L << datacenterIdBits);
    /**
     * 毫秒内自增位
     */
    private final long sequenceBits = 12L;
    private final long workerIdShift = sequenceBits;
    private final long datacenterIdShift = sequenceBits + workerIdBits;
    /**
     * 时间戳左移动位
     */
    private final long timestampLeftShift = sequenceBits + workerIdBits + datacenterIdBits;
    private final long sequenceMask = -1L ^ (-1L << sequenceBits);

    private final long workerId;

    /**
     * 数据标识 ID 部分
     */
    private final long datacenterId;
    /**
     * 并发控制
     */
    private long sequence = 0L;
    /**
     * 上次生产 ID 时间戳
     */
    private long lastTimestamp = -1L;
	
    /**
    * 初始化 数据中心id和当前机器id
    *
    */
    public Sequence() {
        this.datacenterId = getDatacenterId(maxDatacenterId);
        this.workerId = getMaxWorkerId(datacenterId, maxWorkerId);
    }

    /**
     * 有参构造器
     *
     * @param workerId     工作机器 ID
     * @param datacenterId 序列号
     */
    public Sequence(long workerId, long datacenterId) {
        Assert.isFalse(workerId > maxWorkerId || workerId < 0,
                String.format("worker Id can't be greater than %d or less than 0", maxWorkerId));
        Assert.isFalse(datacenterId > maxDatacenterId || datacenterId < 0,
                String.format("datacenter Id can't be greater than %d or less than 0", maxDatacenterId));
        this.workerId = workerId;
        this.datacenterId = datacenterId;
    }

    /**
     * 获取 maxWorkerId
     */
    protected static long getMaxWorkerId(long datacenterId, long maxWorkerId) {
        StringBuilder mpid = new StringBuilder();
        mpid.append(datacenterId);
        String name = ManagementFactory.getRuntimeMXBean().getName();
        if (StringUtils.isNotBlank(name)) {
            /*
             * GET jvmPid
             */
            mpid.append(name.split(StringPool.AT)[0]);
        }
        /*
         * MAC + PID 的 hashcode 获取16个低位
         */
        return (mpid.toString().hashCode() & 0xffff) % (maxWorkerId + 1);
    }

    /**
     * 数据标识id部分
     */
    protected static long getDatacenterId(long maxDatacenterId) {
        long id = 0L;
        try {
            InetAddress ip = InetAddress.getLocalHost();
            NetworkInterface network = NetworkInterface.getByInetAddress(ip);
            if (network == null) {
                id = 1L;
            } else {
                byte[] mac = network.getHardwareAddress();
                if (null != mac) {
                    id = ((0x000000FF & (long) mac[mac.length - 2]) | (0x0000FF00 & (((long) mac[mac.length - 1]) << 8))) >> 6;
                    id = id % (maxDatacenterId + 1);
                }
            }
        } catch (Exception e) {
            logger.warn(" getDatacenterId: " + e.getMessage());
        }
        return id;
    }

    /**
     * 获取下一个 ID
     *
     * @return 下一个 ID
     */
    public synchronized long nextId() {
        long timestamp = timeGen();
        //闰秒
        if (timestamp < lastTimestamp) {
            long offset = lastTimestamp - timestamp;
            if (offset <= 5) {
                try {
                    wait(offset << 1);
                    timestamp = timeGen();
                    if (timestamp < lastTimestamp) {
                        throw new RuntimeException(String.format("Clock moved backwards.  Refusing to generate id for %d milliseconds", offset));
                    }
                } catch (Exception e) {
                    throw new RuntimeException(e);
                }
            } else {
                throw new RuntimeException(String.format("Clock moved backwards.  Refusing to generate id for %d milliseconds", offset));
            }
        }

        if (lastTimestamp == timestamp) {
            // 相同毫秒内，序列号自增
            sequence = (sequence + 1) & sequenceMask;
            if (sequence == 0) {
                // 同一毫秒的序列数已经达到最大
                timestamp = tilNextMillis(lastTimestamp);
            }
        } else {
            // 不同毫秒内，序列号置为 1 - 3 随机数
            sequence = ThreadLocalRandom.current().nextLong(1, 3);
        }

        lastTimestamp = timestamp;

        // 时间戳部分 | 数据中心部分 | 机器标识部分 | 序列号部分
        return ((timestamp - twepoch) << timestampLeftShift)
                | (datacenterId << datacenterIdShift)
                | (workerId << workerIdShift)
                | sequence;
    }

    protected long tilNextMillis(long lastTimestamp) {
        long timestamp = timeGen();
        while (timestamp <= lastTimestamp) {
            timestamp = timeGen();
        }
        return timestamp;
    }

    protected long timeGen() {
        return SystemClock.now();
    }

}
```
