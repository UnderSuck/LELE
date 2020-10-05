# CentOS7 安装 ELK 简明文档

###  **由于本人在安装和使用elk的过程中看了网上大量的文档而踩坑无数，遂把自己心得总结如下，做成简明文档供有需要之人。如果发现什么问题欢迎和我交流。**

### 准备工作

ELK简介
Logstash 对各个服务的日志进行采集、过滤、推送。
Elasticsearch 存储 Logstash 传送的结构化数据，提供给 Kibana 。
Kibana 提供用户 UIweb 页面进行，数据展示和分析形成图表等。

操作系统：CentOS7 安装模式如下图。（推荐）
IP:192.168.31.200
如果是内网使用，安装好系统之后可以关闭防火墙，并且关闭开机启动。

```bash
systemctl stop firewalld
systemctl disable firewalld
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201006000547296.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMwODE0OTg3,size_16,color_FFFFFF,t_70#pic_center)

**软件准备 (以7.9.1版本为例,软件版本需统一.)
下载以下软件到服务器.**

> https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.9.1-x86_64.rpm
> https://artifacts.elastic.co/downloads/logstash/logstash-7.9.1.rpm
> https://artifacts.elastic.co/downloads/kibana/kibana-7.9.1-x86_64.rpm

p.s

```bash
mkdir /download
cd /download
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.9.1-x86_64.rpm
wget https://artifacts.elastic.co/downloads/logstash/logstash-7.9.1.rpm
wget https://artifacts.elastic.co/downloads/kibana/kibana-7.9.1-x86_64.rpm
```

国内下载较慢,可以选择用迅雷下载到PC，然后使用SFTP上传到服务器。

### 开始安装
首先安装java环境

```bash
yum -y install java-1.8.0-openjdk.x86_64
```

### **安装E.L.K.**

```bash
rpm -ivh elasticsearch-7.9.1-x86_64.rpm
rpm -ivh logstash-7.9.1.rpm
rpm -ivh kibana-7.9.1-x86_64.rpm
```


> p.s 使用rpm安装会自动创建好service，使用这种安装方式配置文件一般在/etc/xxx/下，
> 程序文件一般在/usr/share/xxx/下。

### **配置elasticsearch**

1.创建elasticsearch所需的目录，并授权。

```bash
mkdir -p /data/elasticsearch
mkdir -p /data/log/elasticsearch
chown -R elasticsearch.elasticsearch /data/elasticsearch
chown -R elasticsearch.elasticsearch /data/log/elasticsearch
```

2.编辑elasticsearch配置

```bash
vi /etc/elasticsearch/elasticsearch.yml
```

3.编辑以下内容

```bash
cluster.name: xxx #群集名称
node.name: node-1 #节点名称
path.data: /data/elasticsearch #数据目录
path.logs: /data/log/elasticsearch #日志目录
network.host: 192.168.31.200 #绑定IP
http.port: 9200 #服务端口
discovery.seed_hosts: ["127.0.0.1"] #集群发现
cluster.initial_master_nodes: ["node-1"] #填写本机节点名，否则设置密码时会出错。
#设定跨域，head插件需要用到以下选项
http.cors.enabled: true
http.cors.allow-origin: "*"
#http.cors.allow-credentials: true
http.cors.allow-headers: Authorization,X-Requested-With,Content-Length,Content-Type
#设置开启xpack
xpack.security.enabled: true
xpack.license.self_generated.type: basic
# 如果是basic license的话需要加入下面这一行，不然的话restart elasticsearch之后会报错。
xpack.security.transport.ssl.enabled: true
```

4.启动elasticsearch并加入自启动
 

```bash
systemctl start elasticsearch 
 systemctl enable elasticsearch		
```

5.查看elasticsearch运行状态

```bash
systemctl status elasticsearch
```

6.查看端口是否正常

```bash
netstat -ntlp
Active Internet connections (only servers)
tcp6       0      0 192.168.31.200:9200     :::*                    LISTEN      1714/java
tcp6       0      0 192.168.31.200:9300     :::*                    LISTEN      1714/java
```

> p.s.正常的话可以看到9200端口和9300端口存在

**修改elasticsearch密码**

```bash
cd /usr/share/elasticsearch/bin/ #进入可执行文件目录
./elasticsearch-setup-passwords interactive #使用交互式密码设置工具
```


设置成功后会看到以下内容

```bash
Enter password for [elastic]:
Reenter password for [elastic]:
Enter password for [apm_system]:
Reenter password for [apm_system]:
Enter password for [kibana_system]:
Reenter password for [kibana_system]:
Enter password for [logstash_system]:
Reenter password for [logstash_system]:
Enter password for [beats_system]:
Reenter password for [beats_system]:
Enter password for [remote_monitoring_user]:
Reenter password for [remote_monitoring_user]:
Changed password for user [apm_system]
Changed password for user [kibana_system]
Changed password for user [kibana]
Changed password for user [logstash_system]
Changed password for user [beats_system]
Changed password for user [remote_monitoring_user]
Changed password for user [elastic]
```


登录elasticsearch
打开浏览器输入
http://192.168.31.200:9200
由于刚才配置开启了xpack，会弹出身份验证对话框。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201006001635983.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMwODE0OTg3,size_16,color_FFFFFF,t_70#pic_center)

输入
用户名：elastic 
密码：刚才所设密码
即可登录。
登录成功后会看到一段包含相关信息的json字串

```bash
{
  "name" : "node-1",
  "cluster_name" : "xxx",
  "cluster_uuid" : "Ns24bgsXRGWVt358Sc-OSg",
  "version" : {
    "number" : "7.9.1",
    "build_flavor" : "default",
    "build_type" : "rpm",
    "build_hash" : "083627f112ba94dffc1232e8b42b73492789ef91",
    "build_date" : "2020-09-01T21:22:21.964974Z",
    "build_snapshot" : false,
    "lucene_version" : "8.6.2",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```


### **配置logstash**

编辑logstash配置文件

```bash
vi /etc/logstash/logstash.yml
```


修改编辑以下内容

```bash
node.name: xxx #节点名称
path.data: /var/lib/logstash #数据目录
pipeline.ordered: auto #如果只有一个pipeline worker，则自动启用订购。
path.config: /etc/logstash/conf.d #配置文件所在目录
config.test_and_exit: true #配置文件检测
path.logs: /var/log/logstash #日志目录
xpack.monitoring.enabled: true
xpack.monitoring.elasticsearch.username: logstash_system
xpack.monitoring.elasticsearch.password: ********* #刚才设置的密码
xpack.monitoring.elasticsearch.hosts: ["http://192.168.31.200:9200"]
```

启动logstash并且加入自启动

```bash
systemctl start logstash
systemctl enable logstash
```

查看状态

```bash
systemctl status logstash
```


### **配置kibana**

编辑kibana配置文件

```bash
vi /etc/kibana/kibana.yml
```


修改编辑以下内容

```bash
server.port: 5601 #端口
server.host: "192.168.31.200" #如开启外网访问请填写0.0.0.0
server.name: "xxx" #服务名
elasticsearch.hosts: ["http://192.168.31.200:9200"] #ES地址
elasticsearch.username: "elastic" #ES用户名
elasticsearch.password: "******"  #ES密码
i18n.locale: "zh-CN" #设置语言为中文
xpack.monitoring.ui.container.elasticsearch.enabled: true
```


启动kibana并加入自启动

```bash
systemctl start kibana
systemctl enable kibana
```

查看状态

```bash
systemctl status kibana
```


查看kibana端口

```bash
netstat -ntlp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 192.168.31.200:5601     0.0.0.0:*               LISTEN      38057/node
tcp6       0      0 192.168.31.200:9200     :::*                    LISTEN      1714/java
tcp6       0      0 192.168.31.200:9300     :::*                    LISTEN      1714/java
```

> p.s.正常的话可以看到9200，9300，5601端口存在

登录kibana
打开浏览器输入
http://192.168.31.200:5601
正常情况下会看到以下画面
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201006001701989.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMwODE0OTg3,size_16,color_FFFFFF,t_70#pic_center)

输入
用户名:elastic
密码:刚才所设置的密码。
即可登录使用



