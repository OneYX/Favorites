## 阅读目录

- [说明](https://www.cnblogs.com/hellxz/p/11288745.html#说明)
- [脚本内容](https://www.cnblogs.com/hellxz/p/11288745.html#脚本内容)
- [参考文章](https://www.cnblogs.com/hellxz/p/11288745.html#参考文章)

## 说明

最近在写Jenkins自动运维的脚本，由于是用的docker，部署的时候启动容器端口号冲突会导致部署失败，用的微服务也不在乎端口什么的，只求部署成功，所以想了很久，参考了一些文章，还有运维大哥的调试，终于实现了一个脚本，分享出来大家看看，为社区添一块砖加一片瓦 :happy:

**本文主题是得到个未被占用的端口号，这个端口号可以被指定区间**

## 脚本内容

```bash
#!/bin/bash
# @Desc 此脚本用于获取一个指定区间且未被占用的随机端口号
# @Author Hellxz <hellxz001@foxmail.com>

PORT=0
#判断当前端口是否被占用，没被占用返回0，反之1
function Listening {
   TCPListeningnum=`netstat -an | grep ":$1 " | awk '$1 == "tcp" && $NF == "LISTEN" {print $0}' | wc -l`
   UDPListeningnum=`netstat -an | grep ":$1 " | awk '$1 == "udp" && $NF == "0.0.0.0:*" {print $0}' | wc -l`
   (( Listeningnum = TCPListeningnum + UDPListeningnum ))
   if [ $Listeningnum == 0 ]; then
       echo "0"
   else
       echo "1"
   fi
}

#指定区间随机数
function random_range {
   shuf -i $1-$2 -n1
}

#得到随机端口
function get_random_port {
   templ=0
   while [ $PORT == 0 ]; do
       temp1=`random_range $1 $2`
       if [ `Listening $temp1` == 0 ] ; then
              PORT=$temp1
       fi
   done
   echo "port=$PORT"
}
get_random_port 1 10000; #这里指定了1~10000区间，从中任取一个未占用端口号
```