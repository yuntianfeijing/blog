---
title: docker mysql
date: 2018-05-29 17:22:04
tags: docker
---
docker mysql

```
docker run -p 3306:3306 --name armdb -v /workspace/dockerdir/mysql/conf.d:/etc/mysql/conf.d -v /workspace/dockerdir/mysql/logs:/logs -v /workspace/dockerdir/mysql/armdb:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=Thunder#123 -d mysql
```

```
Error:ERROR 1130 (HY000): Host '172.17.0.1' is not allowed to connect to this MySQL server
docker exec -it armdb /bin/bash
root@ce7c9e069925:/#　mysql -uroot -h127.0.0.1
mysql> use mysql;
mysql> update user set host = '%' where user = 'root';
flush privileges;
select host, user from user;

此时可以用mysql -uroot -h127.0.0.1登录，但是没有密码
docker exec -it armdb /bin/bash
mysqladmin -uroot -h127.0.0.1  password Thunder#123
```
