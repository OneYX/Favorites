# [Tomcat nginx log日志按天分割切割](https://www.cnblogs.com/jonnyan/p/9389513.html)

> 利用 Linux 自带的 logrotate 工具来实现按天切割日志.下方已 centos 7 系统为例来实践讲解.

### 原理

**Logrotate是基于CRON来运行的，其脚本是/etc/cron.daily/logrotate，日志轮转是系统自动完成的。**

1. 每晚 cron 后台执行`/etc/cron.daily/`目录下的任务
2. 这会触发`/etc/cron.daily/logrotate`文件，通常这在 linux 安装的时候包含了。 它会执行命令 `/etc/cron.daily/logrotate`
3. `/etc/logrotate.conf` 这个配置文件包含了所有在`/etc/logrotate.d/`的脚本。
4. 这会触发我们下面写的 `/etc/logrotate.d/tomcat-log-cut` 文件。

### Tomcat 日志分割脚本(按天)

- 1.在`/etc/logrotate.d/`下面新建一个文件,**文件名随意**,此处以 tomcat-log-cut 为例.
  `vim /etc/logrotate.d/tomcat-log-cut`
- 2.脚本内容

```
/opt/web/apache-tomcat/logs/catalina.out { #此处的 tomcat 路径请自行更换为你自己的

daily     # 每天切割
rotate 5    # 保留 5 个备份
dateext    # 这个参数很重要！就是切割后的日志文件以当前日期为格式结尾，如 xxx.log-20131216 这样,如果注释掉,切割出来是按数字递增,即 xxx.log-1 这种格式
dateformat .%Y%m%d    # 配合 dateext 使用，紧跟在下一行出现，定义文件切割后的文件名，必须配合 dateext 使用，只支持 %Y %m %d %s 这四个参数
#size 100M    # 大小到达 size 开始转存
notifempty    # 如果是空文件的话，不转储
missingok    # 在日志轮循期间，任何错误将被忽略，例如“文件无法找到”之类的错误。
copytruncate    # 拷贝原日志文件，并且将其变成大小为0的文件。
}
```

- 3.重启 tomcat 服务.
- 4.执行下面的命令测试一下.

```
/usr/sbin/logrotate /etc/logrotate.conf
```

- 5.如果没有成功，再执行下面的命令排查问题

```
/usr/sbin/logrotate -v -f -d /etc/logrotate.d/tomcat-log-cut
```

------

### logrotate 参数说明

| 参数                     | 说明                                                         |
| ------------------------ | ------------------------------------------------------------ |
| compress                 | #通过gzip 压缩转储以后的日志                                 |
| nocompress               | #不做gzip压缩处理                                            |
| create mode owner group  | #轮转时指定创建新文件的属性，如create 0777 nobody nobody     |
| nocreate                 | #不建立新的日志文件                                          |
| delaycompress            | #和compress 一起使用时，转储的日志文件到下一次转储时才压缩   |
| nodelaycompress          | #覆盖 delaycompress 选项，转储同时压缩。                     |
| missingok                | #如果日志丢失，不报错继续滚动下一个日志                      |
| ifempty                  | #即使日志文件为空文件也做轮转，这个是logrotate的缺省选项。   |
| notifempty               | #当日志文件为空时，不进行轮转                                |
| mail address             | #把转储的日志文件发送到指定的E-mail 地址                     |
| olddir directory         | #转储后的日志文件放入指定的目录，必须和当前日志文件在同一个文件系统 |
| noolddir                 | #转储后的日志文件和当前日志文件放在同一个目录下              |
| sharedscripts            | #运行postrotate脚本，作用是在所有日志都轮转后统一执行一次脚本。如果没有配置这个，那么每个日志轮转后都会执行一次脚本 |
| prerotate                | #在logrotate转储之前需要执行的指令，例如修改文件的属性等动作；必须独立成行 |
| postrotate               | #在logrotate转储之后需要执行的指令，例如重新启动 (kill -HUP) 某个服务！必须独立成行 |
| daily                    | #指定转储周期为每天                                          |
| weekly                   | #指定转储周期为每周                                          |
| monthly                  | #指定转储周期为每月                                          |
| rotate count             | #指定日志文件删除之前转储的次数，0 指没有备份，5 指保留5 个备份 |
| dateext                  | #使用当期日期作为命名格式                                    |
| dateformat .%s           | #配合dateext使用，紧跟在下一行出现，定义文件切割后的文件名，必须配合dateext使用，只支持 %Y %m %d %s 这四个参数 |
| size(或minsize) log-size | #当日志文件到达指定的大小时才转储，log-size能指定bytes(缺省)及KB (sizek)或MB(sizem). |

当日志文件 >= log-size 的时候就转储。 以下为合法格式：（其他格式的单位大小写没有试过）
size = 5 或 size 5 （>= 5 个字节就转储）
size = 100k 或 size 100k
size = 100M 或 size 100M

**值得注意的一个配置是：copytruncate **

copytruncate 如果没有这个选项的话，操作方式：是将原log日志文件，移动成类似log.1的旧文件， 然后创建一个新的文件。 如果设置了，操作方式：拷贝原日志文件，并且将其变成大小为0的文件。

区别是：如果进程,比如 Tomcat 使用了一个文件写日志，没有 copytruncate 的话，切割日志时， 把旧日志 Catalina.out->Catalina.out.1 ，然后创建新日志Catalina.out。这时候 tomcat 打开的文件描述符依然时Catalina.out.1，由于没有信号通知tomcat 要换日志描述符，所以它会继续向 Catalina.out.1 写日志，这样就不符合我们的要求了。 因为我们想切割日志后，tomcat 自动会向新的 Catalina.out 文件写日志，而不是旧的 Catalina.out.1 文件.

**解决方法有两个：**

1. 向上面的 tomcat 切割日志配置，再 postrotate 里面写个脚本

postrotate # 在logrotate转储之后需要执行的指令，例如重新启动 (kill -HUP) 某个服务！必须独立成行

```
    [ -s /run/tomcat.pid ] && kill -USR1 `cat /run/tomcat.pid`

endscript
```

这样就是发信号给tomcat ,让 tomcat 关闭旧日志文件描述符，重新打开新的日志文件描述，并写入日志

2.使用 copytruncate 参数，向上面说的，配置了它以后，操作方式是把 Catalina.out 复制一份 成为 Catalina.out.1，然后清空 log 的内容，使大小为0，那此时 log 依然时原来的旧 log，对进程（tomcat）来说，依然打开的是原来的文件描述符，可以继续往里面写日志，而不用发送信号给 tomcat

copytruncate 这种方式操作的时候， 拷贝和清空之间有一个时间差，可能会丢失部分日志数据。

nocopytruncate 备份日志文件不过不截断。

本文参考:https://www.cnblogs.com/276815076/p/7053640.html(主要) 与 https://blog.csdn.net/seelye/article/details/79276216.做了整理优化.