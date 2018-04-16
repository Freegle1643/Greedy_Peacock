# Greedy_Peacock

A.K.A. G.P.



## 计划的环境准备

使用Windows 10下的Ubuntu子系统来进行环境的搭建，使用Python 3.5和Django 1.11版本，建立一个虚拟环境，在里面进行编程，连接远端的MySQL数据库

```bash
sudo apt-get install python3
wget https://bootstrap.pypa.io/get-pip.py
sudo python3 get-pip.py
sudo apt-get install mysql-client-core-5.7
```

创建虚拟环境	

```bash
cd /mnt/d/path/to/project
sudo pip3 install virtualenv
virtualenv gpsite
```

进如`gpsite`使用`ls`可以看到有如下的工作目录

```bash
bin  include  lib
```

进入`bin`进行激活

```bash
cd bin
source ./activate
```

安装Django

```bash
sudo pip3 install django==1.11
sudo pip3 install mysqlclient
```

## 事实上的环境准备 Part 1

大概是因为之前的一些莫名其妙的配置，最终投奔了宇宙最费电IDE系列PyCharm，同时Navicat连上数据库后创建的表也不能被看到，只有使用命令行或者MySQL Workbench创建的表才能被真正地使用。

### 更改数据库为MySQL

进入`settings.py`，将如下的代码更改为MySQL连接的配置

```python
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': '***',
        'USER': 'root',
        'PASSWORD': '***',
        'HOST': '118.***.***.***',
    }
```

运行服务器，检查是否能看到Django的默认启动界面

## 数据库表的设计

- NormalUser

与Django自带的用户表分隔开，用户的登陆和认证使用自带的表，我们的显示等方面更丰富的使用

```
user_id INT
user_name CHAR
1)user_ip INT 不同的IP就进行异地提醒
2)user_ip CHAR 不同的城市再进行异地提醒 （MySQL中使用INET_NTOA()进行转换）
user_con_sum INT 容器数量
```

使用OneToOneField和Django自带User相连

[在MySQL中存储IP](https://stackoverflow.com/questions/2542011/most-efficient-way-to-store-ip-address-in-mysql?utm_medium=organic&utm_source=google_rich_qa&utm_campaign=google_rich_qa)

[使用ipip的地址库来进行IP归属地查询](https://www.ipip.net/api.html)

- NodeInfo

Docker节点服务器的表，用来管理负载均衡

```
node_id INT
node_ip INT
node_name CHAR
node_core SMALLINT（居然contains error，那么就int吧）
node_mem INT 总内存大小，精确到MB（使用free -m）
node_avmem INT 可用内存大小，可以计算出哪个负载更高
node_storage INT 由于并不能控制container使用的空间大小，我们仅作存储，精确到GB（使用df -h）
node_avstorage INT 可用存储空间
node_os CHAR 
node_con_sum INT
```

[查看Disk Space的Linux命令](https://www.tecmint.com/how-to-check-disk-space-in-linux/)

[查看Memory使用的Linux命令](https://www.binarytides.com/linux-command-check-memory-usage/)

[可用于查看容器数量的命令](https://stackoverflow.com/questions/38459853/how-can-i-check-how-many-containers-are-running-a-certain-docker-image?utm_medium=organic&utm_source=google_rich_qa&utm_campaign=google_rich_qa)：

```bash
haoyuan@haoyuan-Broadwell:~$ docker ps -a | wc -l
38
haoyuan@haoyuan-Broadwell:~$ docker ps | wc -l
3
```

很神奇的是会多算一个

另外也可以使用`docker info`来计算，只是需要对输出结果进行一些处理，因为输出太多了

```bash
haoyuan@haoyuan-Broadwell:~$ docker info
Containers: 37
 Running: 2
 Paused: 0
 Stopped: 35
Images: 19
Server Version: 18.03.0-ce
...
```

- ImageItem

每个镜像都会是一条记录

```
img_id INT
img_repo CHAR
img_tag CHAR
img_imid CHAR 虽然使用repo和tag就可以知道是哪一个image，但是image id可以使用一个字段就读出
img_size INT MB
*img_sum INT 有多少容器是这个image 好像不好搞，可以通过每创建一个这个镜像的容器，就+1来实现？【见上文】
```

- ContainerItem

每个容器都会是一条记录

```
con_id INT
con_name CHAR
con_image INT 外键，指向Image的主键
con_user INT 外键，指向NormalUser主键
con_port SMALLINT 
con_cid CHAR 对应的Docker管理的id
con_perform TINYINT 1，2，3几个档次档次（最多支持255档次），代表1核1G，1核2G，2核1G，2核2G...
con_stat TINYINT 0 created（使用Docker create或者run没有成功）1 up，即已启动 2 exited，即已退出 
*con_size 实际是没有意义的，因为容器的层叠式架构意味着每个容器所需要的功能大多已由镜像提供，自己的modification其实并不很多，可以用docker system df -v 说明
```

[Integer Types (Exact Value) - INTEGER, INT, SMALLINT, TINYINT, MEDIUMINT, BIGINT](https://dev.mysql.com/doc/refman/5.7/en/integer-types.html)

## 事实上的环境准备 Part 2

### 配置遗留数据库

- 从已设计好的表中生成模型

```bash
python manage.py inspectdb > ./remotop/models.py
```

- 对Django的核心模块进行迁移

```bash
python manage.py migrate
```

