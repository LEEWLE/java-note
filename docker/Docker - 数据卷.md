### 数据卷

数据卷是一个可供一个或多个容器使用的特殊目录，它绕过文件系统，可以提供很多有用的特性：

- 数据卷可以在容器之间共享和重用
- 对数据卷的修改会立马生效
- 对数据卷的更新，不会影响镜像
- 数据卷默认会一直存在，即使容器被删除

#### 容器内创建一个数据卷

使用 docker run 命令时，加上 -v 参数可以在容器内创建一个数据卷，多次使用可以创建多个数据卷

```shell
# -P 参数是允许外部访问容器需要暴露的端口
# web 是容器名
# /webapp 是挂载的目录
# training/webapp 是镜像
# python app.py 是执行的命令
docker run -d -P --name web -v /webapp training/webapp python app.py
```

以上命令是使用 taining/webapp 镜像创建一个 web 容器，并创建一个数据卷挂载到容器的 /webapp 目录

#### 挂载一个主机目录作为数据卷

```sh
# /src/webapp  是主机的本地目录，该路径必须为绝对路径
# /opt/webapp  是容器的目录，如果该目录不存在，Docker 会自动创建
docker run -d -P --name web -v /src/webapp:/opt/webapp training/webapp python app.py
```

查看挂载信息

```shell
[root@VM-0-14-centos ~]# docker inspect -f {{.Mounts}} 8b55cbd60d36ba5
[{bind  /src/webapp /opt/webapp   true rprivate}]
```

注意：Docker 挂载数据卷默认权限是读写，可加参数 ro 指定为只读

```
docker run -d -P --name web -v /src/webapp:/opt/webapp:ro training/webapp python app.py
```

#### 挂载一个本地文件作为数据卷

```shell
# ~/.bash_history 该文件的作用是保存了当前用户使用过的历史命令
# 在容器中查看 /.bash_history 文件，会发现内容和 ~/.bash_history 文件相同
docker run --rm -it -v ~/.bash_history:/.bash_history centos:7 /bin/bash
```

注意：挂载文件到容器，使用 vi 或 sed 等文件编辑工具，可能造成文件 inode 改变，从 Docker 1.1.0 起，这会导致报错信息

### 数据卷容器

数据卷容器可以在容器之间共享一些持续更新的数据

**案例：数据卷容器的使用**

创建数据卷容器 dbdata，并在其中创建一个数据卷挂载到 /dbdata

```
docker run -it -v /dbdata --name dbdata centos:7
```

创建两个容器 db1 和 db2，并从 dbdata 容器挂载数据卷

```shell
docker run -it --volumes-from dbdata --name db1 centos:7
docker run -it --volumes-from dbdata --name db2 centos:7
```

挂载成功后，在任何一个容器中写入内容，其他的容器也都会存在

如果删除了挂载的容器（dbdata、db1、db2），数据卷并不会被自动删除、如果要删除一个数据卷，必须在删除最后一个挂载它的容器时显示的使用 `docker rm -v ` 指定同时删除管理的容器

### 数据卷容器迁移数据

#### 备份

```shell
# --volumes-from dbdata: 容器 worker 挂载 dbdata 容器的数据卷
# -v $(pwd):/backup：挂载本地的当前目录到 worker 容器的 /backup 目录
# tar cvf /backup/backup.tar /dbdata ：打包文件，即当前目录下的 backup.tar
docker run --volumes-from dbdata -v $(pwd):/backup --name worker centos:7 tar cvf /backup/backup.tar /dbdata
```

#### 恢复

首先创建数据卷容器 dbdata3

```shell
docker run -itd -v /dbdata --name dbdata3 centos:7 /bin/bash
```

创建另一个容器，挂载 dbdata3 容器，并解压备份文件到所挂载的容器卷中

```shell
docker run --volumes-from dbdata3 -v $(pwd):/backup --name bb1 centos:7 tar xvf /backup/backup.tar 
```

最后进入数据卷容器中的 /dbdata 目录下会有解压文件存在

```shell
[root@VM-0-14-centos ~]# docker attach dbdata3
[root@51e10bed0b41 /]# ls
anaconda-post.log  bin  dbdata  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
[root@51e10bed0b41 /]# cd dbdata/
[root@51e10bed0b41 dbdata]# ls
text
```

