title: "MySQL ERROR 1045 (28000): Access denied for user 'dev'@'localhost' (using password: YES)"
date: 2016-10-11 22:00:30
tags: debug mysql
---
新安装mysql，本机用mysql -udev -p -P3306 -h127.0.0.1登陆时，会报错
MySQL ERROR 1045 (28000): Access denied for user 'dev'@'localhost' (using password: YES)

#####原因如下：

因为存在匿名而且host为localhost的用户，本机登陆时，没有指定/S socket的情况下，会使用tcp连接，mysql匹配规则会先匹配排序过的host和username
在host都是localhost的情况下，匿名用户会"遮盖"你的任意用户，比如“[any_username]'@'%”
"'dev'@'localhost'会先匹配，“''@'%'”而不是"'dev'@'%'

#####解决方式：
1. 使用socket方式连接，如 mysql -udev -p123 -P3306 -S/tmp/socket1.mysql
2. 删除匿名用户 ,例如 “''@'%'”


以下引用自stackoverflow

You probably have an anonymous user ''@'localhost' or ''@'127.0.0.1'.

As per the manual:

> When multiple matches are possible, the server must determine which of them to use. 
> It resolves this issue as follows: (...) When a client 
> attempts to connect, the server looks through the rows [of table
> mysql.user] in sorted order. The server uses the first row that
> matches the client host name and user name. (...) The server uses
> sorting rules that order rows with the most-specific Host values
> first. Literal host names [such as 'localhost'] and IP addresses are
> the most specific. Hence, such an anonymous user would "mask" any
> other user like '[any_username]'@'%' when connecting from localhost.

'bill'@'localhost' does match 'bill'@'%', but would match (e.g.)
''@'localhost' beforehands.

The recommended solution is to drop this anonymous user (this is
usually a good thing to do anyways).

Below edits are mostly irrelevant to the main question. These are only
meant to answer some questions raised in other comments within this
thread.

Edit 1

Authenticating as 'bill'@'%' through a socket.


    root@myhost:/home/mysql-5.5.16-linux2.6-x86_64# ./mysql -ubill -ppass --socket=/tmp/mysql-5.5.sock
    Welcome to the MySQL monitor (...)

    mysql> SELECT user, host FROM mysql.user;
    +------+-----------+
    | user | host      |
    +------+-----------+
    | bill | %         |
    | root | 127.0.0.1 |
    | root | ::1       |
    | root | localhost |
    +------+-----------+
    4 rows in set (0.00 sec)

    mysql> SELECT USER(), CURRENT_USER();
    +----------------+----------------+
    | USER()         | CURRENT_USER() |
    +----------------+----------------+
    | bill@localhost | bill@%         |
    +----------------+----------------+
    1 row in set (0.02 sec)

    mysql> SHOW VARIABLES LIKE 'skip_networking';
    +-----------------+-------+
    | Variable_name   | Value |
    +-----------------+-------+
    | skip_networking | ON    |
    +-----------------+-------+
    1 row in set (0.00 sec)

Edit 2

Exact same setup, except I re-activated networking, and I now create
an anonymous user ''@'localhost'.


    root@myhost:/home/mysql-5.5.16-linux2.6-x86_64# ./mysql
    Welcome to the MySQL monitor (...)

    mysql> CREATE USER ''@'localhost' IDENTIFIED BY 'anotherpass';
    Query OK, 0 rows affected (0.00 sec)

    mysql> Bye

    root@myhost:/home/mysql-5.5.16-linux2.6-x86_64# ./mysql -ubill -ppass \
        --socket=/tmp/mysql-5.5.sock
    ERROR 1045 (28000): Access denied for user 'bill'@'localhost' (using password: YES)
    root@myhost:/home/mysql-5.5.16-linux2.6-x86_64# ./mysql -ubill -ppass \
        -h127.0.0.1 --protocol=TCP
    ERROR 1045 (28000): Access denied for user 'bill'@'localhost' (using password: YES)
    root@myhost:/home/mysql-5.5.16-linux2.6-x86_64# ./mysql -ubill -ppass \
        -hlocalhost --protocol=TCP
    ERROR 1045 (28000): Access denied for user 'bill'@'localhost' (using password: YES)

Edit 3

Same situation as in edit 2, now providing the anonymous user's
password.


    root@myhost:/home/mysql-5.5.16-linux2.6-x86_64# ./mysql -ubill -panotherpass -hlocalhost
    Welcome to the MySQL monitor (...)

    mysql> SELECT USER(), CURRENT_USER();
    +----------------+----------------+
    | USER()         | CURRENT_USER() |
    +----------------+----------------+
    | bill@localhost | @localhost     |
    +----------------+----------------+
    1 row in set (0.01 sec)

Conclusion 1, from edit 1: One can authenticate as 'bill'@'%'through a
socket.

Conclusion 2, from edit 2: Whether one connects through TCP or through
a socket has no impact on the authentication process (except one
cannot connect as anyone else but 'something'@'localhost' through a
socket, obviously).

Conclusion 3, from edit 3: Although I specified -ubill, I have been
granted access as an anonymous user. This is because of the "sorting
rules" advised above. Notice that in most default installations, a
no-password, anonymous user exists (and should be secured/removed).




