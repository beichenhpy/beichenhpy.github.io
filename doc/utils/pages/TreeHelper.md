# 【工具类】查询树形结构
## 表结构如下:
```MySQL
CREATE TABLE `tree` (
  `id` int NOT NULL,
  `parent_id` int DEFAULT NULL,
  `name` varchar(100) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;


INSERT INTO tree values (1,0,'一级目录-1');
INSERT INTO tree values (2,1,'二级目录-1');
INSERT INTO tree values (3,1,'二级目录-1');
INSERT INTO tree values (4,1,'二级目录-1');
INSERT INTO tree values (5,2,'二级目录-2');
INSERT INTO tree values (6,2,'二级目录-2');
INSERT INTO tree values (7,2,'二级目录-2');
```
## 工具类如下：
```java
package cn.beichenhpy.util;

import cn.beichenhpy.enums.SqlConstant;
import com.baomidou.mybatisplus.annotation.IdType;
import com.baomidou.mybatisplus.annotation.TableField;
import com.baomidou.mybatisplus.annotation.TableId;
import com.baomidou.mybatisplus.core.conditions.query.QueryWrapper;
import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import lombok.Data;
import org.springframework.beans.factory.annotation.Autowired;

import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.stream.Collectors;

/**
 * @author beichenhpy
 * @version 1.0.0
 * @apiNote M mapper T 为Tree的子类 用于查询树状层级结构
 * <br>重要：1. 数据库对应的parentId 和 id 需要为int类型 2. 层数为从上到下
 * <br> 不足：需要手动配置bean，不然统一配置会无法注入泛型
 * @see Tree 树数据结构
 * @since 2021/7/24 12:53
 */
public class TreeHelper<T extends TreeHelper.Tree, M extends BaseMapper<T>> {

    @Autowired
    M mapper;

    //为内存计算使用，查询出来的所有树信息,线程安全
    private volatile List<T> allRows;

    /**
     * 双检锁赋值
     */
    public void init() {
        if (allRows == null) {
            synchronized (TreeHelper.class) {
                if (allRows == null) {
                    allRows = new CopyOnWriteArrayList<>(mapper.selectList(null));
                }
            }
        }
    }

    /**
     * 通过api新增层级时调用，刷新成员变量AllRows
     */
    public void updateCache() {
        init();
    }

    /**
     * 缺省的方法查询树形结构 默认使用一次加载到内存
     *
     * @param floor 层数 从上到下
     * @return 返回树形结构
     */
    public List<T> getTree(Integer floor) {
        if (floor <= 0) {
            throw new IllegalArgumentException("floor must > 0");
        }
        int parentId = floor - 1;
        init();
        //重置当前层数
        return getChildren(parentId);
    }


    /**
     * @param parentId 父极目录id
     * @return 整个树
     */
    private List<T> getChildren(Integer parentId) {
        List<T> trees;
        int floor = parentId;
        trees = allRows.stream()
                .filter(t -> parentId.equals(t.getParentId()))
                .collect(Collectors.toList());
        if (!trees.isEmpty()) {
            floor++;
            for (T tree : trees) {
                tree.setCurrentFloorNum(floor);
                //递归查询，直到return null结束
                tree.setChildren(getChildren(tree.getId()));
            }
            return trees;
        }
        //未查询到结束递归
        return null;
    }

    /**
     * @author beichenhpy
     * @version 1.0.0
     * @apiNote 树及结构 子类可进行扩充
     * @since 2021/7/24 12:57
     */
    @Data
    public static class Tree {
        @TableId(type = IdType.AUTO)
        private Integer id;
        private Integer parentId;
        /**
         * 当前目录层级
         */
        @TableField(exist = false)
        private Integer currentFloorNum;
        /**
         * 叶子节点，（下一级）
         */
        @TableField(exist = false)
        private List<? extends TreeHelper.Tree> children;
    }

    /**
     * 无缓存方式
     */
    public static class TreeHelperNoCache {

        /**
         * 缺省的方法查询树形结构 默认使用一次加载到内存
         *
         * @param floor 层数 从上到下
         * @return 返回树形结构
         */
        public static <T extends TreeHelper.Tree, M extends BaseMapper<T>> List<T> getTree(Integer floor, M mapper) {
            if (floor <= 0) {
                throw new IllegalArgumentException("floor must > 0");
            }
            int parentId = floor - 1;
            //重置当前层数
            return getChildren(parentId, mapper.selectList(new QueryWrapper<T>().ge(SqlConstant.PARENT_ID.getValue(), parentId)));
        }


        /**
         * @param parentId 父极目录id
         * @return 整个树
         */
        private static <T extends TreeHelper.Tree> List<T> getChildren(Integer parentId, List<T> rows) {
            List<T> trees;
            int floor = parentId;
            trees = rows.stream()
                    .filter(t -> parentId.equals(t.getParentId()))
                    .collect(Collectors.toList());
            if (!trees.isEmpty()) {
                floor++;
                for (T tree : trees) {
                    tree.setCurrentFloorNum(floor);
                    //递归查询，直到return null结束
                    tree.setChildren(getChildren(tree.getId(), rows));
                }
                return trees;
            }
            //未查询到结束递归
            return null;
        }
    }
}
```
## 使用方式
1. 无缓存，提供静态方式
   
新建一个子类继承于`TreeHelper.Tree`

`TreeHelper`
```java
package cn.beichenhpy.modal;

import cn.beichenhpy.util.TreeHelper;
import com.baomidou.mybatisplus.annotation.TableName;
import lombok.*;

/**
 * @author beichenhpy
 * @version 1.0.0
 * @apiNote
 * @since 2021/7/24 13:43
 */
@ToString(callSuper = true)
@EqualsAndHashCode(callSuper = true)
@TableName("tree")
@Data
public class TreeInfo extends TreeHelper.Tree{
    private String name;
}
```
`Controller`
```java
package cn.beichenhpy.controller;

import cn.beichenhpy.mapper.TreeMapper;
import cn.beichenhpy.modal.TreeInfo;
import cn.beichenhpy.util.TreeHelper;
import lombok.AllArgsConstructor;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

@AllArgsConstructor
@RestController
@RequestMapping("/api/v1")
public class ConsumerController {

    private final TreeMapper treeMapper;
 

    @GetMapping("/tree")
    public List<TreeInfo> getTree(){
        // 无缓存
        return TreeHelper.TreeHelperNoCache.getTree(treeMapper,1);
    }
}
```
2. 有缓存，需要注册bean
   
`注册bean`
```java
@Bean
public TreeHelper<TreeInfo, TreeMapper> treeHelper(){
    return new TreeHelper<>();
}
```
`Test`
```java
@Autowired
TreeHelper<TreeInfo,TreeMapper> treeHelper;
@Autowired
TreeMapper treeMapper;
@Test
public void test(){
    long start1 = System.currentTimeMillis();
    List<TreeInfo> tree2 = treeHelper.getTree(3);
    long end1 = System.currentTimeMillis() - start1;
    log.info("me:cost:{}",end1);
    log.info("result:me:{}",tree2);
    long start = System.currentTimeMillis();
    List<TreeInfo> tree = treeHelper.getTree(3);
    long end = System.currentTimeMillis() - start;
    log.info("me:cost:{}",end);
    log.info("result:me:{}",tree);
}
```
