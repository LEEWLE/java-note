### 基础网络配置

#### 端口映射实现容器访问

**从外部访问容器应用**

在启动容器时，若不指定对应参数，在容器外部是无法通过网络来访问容器内的网络应用和服务

可以使用 -P 或 -p 参数来指定端口的映射，当使用 -P 时，Docker 会随机映射一个 49000 ~ 49900 的端口至容器内部开放的网络端口

-p 可以指定端口的映射，一个指定端口上只能绑定一个容器。支持格式有

`ip:主机端口:容器端口 `：映射到指定地址的任意端口

`ip::容器端口 `：映射到指定地址的任意端口

`容器端口:主机端口`：映射所有接口地址

**实例：随机分配给 webapp 端口**

```shell
docker run -d -P training/webapp python app.py
```

查看分配端口信息，通过 `docker ps -l 或 docker logs -f '容器'`

```
[root@VM-0-14-centos ~]# docker ps -l
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                     NAMES
c2e20d9bedea        training/webapp     "python app.py"     2 minutes ago       Up 2 minutes        0.0.0.0:32773->5000/tcp   sleepy_bell
```

根据上述命令可以看到 本地主机的 32773 被映射到了容器的 5000 端口

**实例：映射所有接口地址**

以下命令默认会绑定本地所有接口上的地址，多次使用可以绑定多个端口

```shell
docker run -d -p 9001:9001 --name pcontainer training/webapp python app.py
```

查看分配端口信息

```shell
[root@VM-0-14-centos ~]# docker ps -l
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                              NAMES
2cab88990eb0        training/webapp     "python app.py"     31 seconds ago      Up 31 seconds       5000/tcp, 0.0.0.0:9001->9001/tcp   pcontainer
```

#### 容器互联

**link 方式互联**

创建一个数据库容器

```shell
docker run -d --name db training/postgres
```

创建一个新的 web 容器，并将它连接到 db 容器

```shell
# --link 参数格式：--link name:alias,其中 name 是链接的容器名称，alias是链接的别名
docker run -d -P --name web --link db:db training/webapp python app.py
```

**查询容器公开连接方式**

**环境变量**

```
docker run --rm --name web2 --link db:db training/webapp env
```

其中DB_开头的环境变量是供web容器连接db容器使用

**查看 /etc/hosts 文件**

Docker 添加了 host 信息到父容器的 /etc/hosts 文件中

```shell
docker run -ti --rm --link db:db training/webapp /bin/bash
```

查看 hosts 文件

```shell
root@7fa28a1608ad:/opt/webapp# cat /etc/hosts
172.17.0.7	db a85abee4e679
172.17.0.8	7fa28a1608ad
```

使用 ping 命令测试是否连通

```shell
root@7fa28a1608ad:/opt/webapp# ping db
PING db (172.17.0.7) 56(84) bytes of data.
64 bytes from db (172.17.0.7): icmp_seq=1 ttl=64 time=0.104 ms
64 bytes from db (172.17.0.7): icmp_seq=2 ttl=64 time=0.081 ms
```

