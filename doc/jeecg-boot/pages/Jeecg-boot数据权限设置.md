参考 [jeecg-boot 数据权限](https://blog.csdn.net/gwcgwcjava/article/details/104162346)


1. 在菜单管理中 选择要设置权限的子菜单  
![pic1](../../../pic/jeecg-boot/data-permission-1.png)  
2. 点击数据规则  
![pic2](../../../pic/jeecg-boot/data-permission-2.png)  
3. 新建一个如图所示的规则（以部门分离为例）
sysOrgCode 右模糊 #{sys_org_code} 这样就可以上级查询下级的信息 
![pic3](../../../pic/jeecg-boot/data-permission-3.png)  
4. 在角色管理中，在你想要的设置部门分离的角色，配置对应的数据权限规则  
![pic4](../../../pic/jeecg-boot/data-permission-4.png)  
5. 点击授权，右侧弹出页面，注意：有红色的代表设置了数据权限规则  
![pic5](../../../pic/jeecg-boot/data-permission-5.png)  
6. 点击红色的部分，弹出右侧框，选择你要启用的数据规则，保存生效  
![pic6](../../../pic/jeecg-boot/data-permission-6.png)  
7. 在后台需要进行数据权限规则生效的方法上，添加注解
```java
// pageComponent为菜单配置的组件位置
@PermissionData(pageComponent = "smt/device/SmtDeviceList")
```


