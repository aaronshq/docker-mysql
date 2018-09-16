本文旨在提供最小可用的启动文档，用于测试验证 MySQL 功能。通过本文你可以了解怎么通过 Docker 快速启动一个 MySQL 容器，并进行 MySQL 的配置和备份功能。

官方文档参考 https://hub.docker.com/_/mysql/

## 运行 MySQL
在本地编排容器最简单的方法就是用 docker-compose，一个配置文件就能把整个容器环境拉起来，配置可以固定下来，不需要每次记忆 docker run 下的配置参数。

编写 `docker-compose.yml` 文件，内容如下：

```
version: '3.1'

services:

  db:
    image: mysql
    ports:
        - "3306:3306"
    command: --default-authentication-plugin=mysql_native_password --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: rootpasswd
      MYSQL_DATABASE: mydb
      MYSQL_USER: myuser
      MYSQL_PASSWORD: mypasswd

    networks:
        localnet:
            aliases:
                - mysql-default

networks:
    localnet:

```

其中 MySQL_XXX 创建了 root 密码和普通用户。localnet 让容器关联到本地网络，可以通过本地 127.0.0.1 或者 localhost 进行访问。

启动 MySQL
```
shell> docker-compose up
```

停止 MySQL
```
shell> docker-compose stop # 如果要彻底删除使用命令down
```

## 连接 MySQL
一般有三种方式：
1. 本地 MySQL 客户端
2. 直接进入容器
3. 通过容器启动一个 MySQL 客户端

方法3由于需要启动另一个容器，不是最简单的操作，但好处是有隔离的环境。这里就不详细说明了，有这个需要的读者可以参考 Docker MySQL 官方文档）。

### 本地 MySQL 客户端
一般来说本地还是建议安装个 MySQL 客户端，安装包可以参考 [MySQL 官方文档](https://dev.mysql.com/downloads/shell/)或者利用各种系统的包管理工具，比如 MacOS下可以通过以下命令安装（这里会把整个 MySQL Server 都一起安装了）
```
brew install mysql
```
其他系统就不展开了。安装好后，直接通过 shell 执行以下命令即可进入 MySQL。
```
shell>mysql -uroot -prootpasswd -h 127.0.0.1
```
### 通过容器直接进入 MySQL
这个方法的原理是 MySQL 容器本身集成了客户端，所以可以通过进入容器的方式连接 MySQL。

首先通过`docker ps`命令找到容器的名称，这里是`mysql_db_1`
```
docker exec -it mysql_db_1 mysql -uroot -prootpasswd
```

## 配置 MySQL
### 配置文件
通过挂载本地目录到容器中的`/etc/mysql/conf.d/`进行配置文件导入，假设以下配置
```
.
├── config
│   └── my_custom.cnf
└── docker-compose.yml
```
配置一个最简单的修改，比如调整端口为3307
```
# config/my_custom.cnf
[mysqld]
port        = 3307
```

修改 docker-compose.yml 文件，在 db 项下面添加配置
```
    db:
    
    # 原配置保留，这里省略......
    
        ports:  
        - "3307:3307"
        volumes:
        - ./config:/etc/mysql/conf.d

```

### 配置参数
docker 的 mysql 镜像入口提供很多常用的 mysql 配置，可以在运行容器时配置参数，比如本文中的几个配置数据库用户和 root 用户的密码。更多配置也可以参考[官网](https://hub.docker.com/_/mysql/)。

### 备份、恢复数据
#### MySQL Dump
可以像操作普通数据库一样进行`mysqldump`进行备份，并通过 mysql 导入数据。

#### 物理备份
将 MySQL 的文件目录映射到本地指定的目录，通过整个目录进行备份，在 volumes 配置中新增一个磁盘映射
```
...
    volumes:
        - ./config:/etc/mysql/conf.d
        - ./datadir:/var/lib/mysql
```
在目录中新建`datadir`目录。

通过`docker-compose down && docker-compose up`重启容器，
即可观察到`datadir`目录中的文件，已经包含了相应的文件。

```
# ls datadir
ib_logfile0  ibdata1  mysql      undo_001
ib_logfile1  ibtmp1   mysql.ibd  undo_002
```

简单测试一下，进入 MySQL 后，创建一个新表
```
CREATE TABLE mytable( id INT );
```

这时关掉重启容器，只要`datadir`的内容没有变，重新进入之后，发现这个表依然在。即可以通过将`datadir`目录达到备份和恢复的目的。现在很多文件系统提供快照功能，利用磁盘快照，实现快速的备份功能。
