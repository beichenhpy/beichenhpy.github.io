参考 [jeecg-boot 数据权限](https://blog.csdn.net/gwcgwcjava/article/details/104162346)


1、在菜单管理中 选择要设置权限的子菜单
![image.png](https://cdn.nlark.com/yuque/0/2021/png/8425268/1615193036485-b6b44d74-df23-4e02-a35a-d63501aded3a.png#align=left&display=inline&height=237&margin=%5Bobject%20Object%5D&name=image.png&originHeight=237&originWidth=1620&size=30090&status=done&style=none&width=1620)
2、点击数据规则
![image.png](https://cdn.nlark.com/yuque/0/2021/png/8425268/1615193173789-46c38942-1968-4e73-8e97-efdf19c559fc.png#align=left&display=inline&height=431&margin=%5Bobject%20Object%5D&name=image.png&originHeight=431&originWidth=623&size=19897&status=done&style=none&width=623)
3、新建一个如图所示的规则（以部门分离为例）
sysOrgCode 右模糊 #{sys_org_code} 这样就可以上级查询下级的信息
![image.png](https://cdn.nlark.com/yuque/0/2021/png/8425268/1615193209620-08f6f5bc-64f9-4b60-8ab9-cb13ff8e428d.png#align=left&display=inline&height=460&margin=%5Bobject%20Object%5D&name=image.png&originHeight=460&originWidth=994&size=20732&status=done&style=none&width=994)
4、在角色管理中，在你想要的设置部门分离的角色，配置对应的数据权限规则
![image.png](https://cdn.nlark.com/yuque/0/2021/png/8425268/1615193311757-fdbf843b-df1b-44d0-b23d-b78f81945e16.png#align=left&display=inline&height=776&margin=%5Bobject%20Object%5D&name=image.png&originHeight=776&originWidth=1889&size=83685&status=done&style=none&width=1889)
5、点击授权，右侧弹出页面，注意：有红色的代表设置了数据权限规则
![image.png](https://cdn.nlark.com/yuque/0/2021/png/8425268/1615193366327-46fed01f-91eb-46cc-92c7-ad4f251185fe.png#align=left&display=inline&height=216&margin=%5Bobject%20Object%5D&name=image.png&originHeight=216&originWidth=658&size=9655&status=done&style=none&width=658)
6、点击红色的部分，弹出右侧框，选择你要启用的数据规则，保存生效
![image.png](https://cdn.nlark.com/yuque/0/2021/png/8425268/1615193384312-7779984d-f54c-4000-b3b7-866015eccac8.png#align=left&display=inline&height=294&margin=%5Bobject%20Object%5D&name=image.png&originHeight=294&originWidth=840&size=22050&status=done&style=none&width=840)
7、在后台需要进行数据权限规则生效的方法上，添加注解
```java
// pageComponent为菜单配置的组件位置
@PermissionData(pageComponent = "smt/device/SmtDeviceList")
```


