**使用环境为Linux-Centos7(阿里云1c2g)**  
参考文章   
[Elasticsearch插件安装之elasticsearch-head](https://blog.51cto.com/11612299/2405051)  
[elasticsearch-head-github](https://github.com/mobz/elasticsearch-head)
## 一、下载Elasticsearch
学习时使用的时Elasticsearch 6.8.12版本

- 点击连接[下载elasticsearch](https://www.elastic.co/cn/downloads/past-releases/elasticsearch-6-8-12)
- 下载后解压到/opt文件夹下
## 二、配置
### 1、环境配置

- 需要jdk环境为 Oracle-jdk-8/open-jdk-8
- nginx ---用于反向代理 ** elasticsearch (es.beichenhpy.cn)**
### 2、jvm.options配置
由于服务器内存很小，所以需要调整`jvm`的配置
文件目录为 **_/opt/elasticsearch-6.8.12/config/jvm.options_**
```
#调整Xms和Xmx大小
-Xms256M
-Xmx256M
#添加配置 GC回收线程数
8-13:-XX:ParallelGCThreads=5
```
### 3、elasticsearch.yml配置
文件目录为 **_/opt/elasticsearch-6.8.12/config/__elasticsearch.__yml**
添加跨域，其他都为默认值
```yaml
http.cors.enabled: true
http.cors.allow-origin: "*"
#跨域
```
## 三、启动
**注意事项：** elasticsearch不允许使用root用户登录，所以可以为其创建一个用户用来登录
### 创建用户
```bash
# 添加组
groupadd es
# 添加用户
useradd es -g es -p es
# 授权
chown -R es:es /opt/es/elasticsearch-6.8.12
```
### 切换es用户启动
```bash
# 切换用户
su es
# 进入bin目录
cd /opt/elasticsearch-6.8.12/bin
# 启动 后台启动
./elasticsearch -d 
```
# Elasticsearch-head
根据github上的地址，可以下载chrome插件，直接使用 [商店地址](https://chrome.google.com/webstore/detail/elasticsearch-head/ffmkiejjmecolpfloofpjologoblkegm/)




