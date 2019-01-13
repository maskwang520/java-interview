#### if用法
判断文件是否存在
```shell
#!/bin/bash
mydir=/home/likegeeks
if [ -d $mydir ]; then
    echo "Directory $mydir exists"
    cd $mydir
    ls
else
    echo "NO such file or directory $mydir"
fi
```
#### loop
找出PATH目录下可执行的文件
```shell
#!/bin/bash
# 文件分割符号
IFS=:
for dir in $PATH; do
    echo "$dir:"
    for myfile in $dir/*; do
        if [ -x $myfile ]; then
            echo " $myfile"
        fi
    done
done
```

#### Option和Read
接受option和param
```shell
total=1
while [ -n "$1" ]; do # while loop starts
    echo "#$total = $1"
    total=$(($total + 1))
    shift
done
```
读取文件的每一行
```shell
#!/bin/bash
count=1
# Get file content then pass to read command by iterating over lines using while command
cat myfile | while read line; do
    echo "#$count: $line"
    count=$(($count + 1))
done
echo "Finished"
```

常见的脚本结构
```shell
#!/usr/bin/env bash

git pull

cd ..

mvn -Dmaven.test.skip=true clean package

# 分发路由
cp /root/work/netty-action/cim-forward-route/target/cim-forward-route-1.0.0-SNAPSHOT.jar /root/work/route/

appname="route" ;
# 使用了管道，上一个的输出作为下一个的输入 grep -v 是inverse，即不取
PID=$(ps -ef | grep $appname | grep -v grep | awk '{print $2}')

# 遍历杀掉 pid 遍历数组
for var in ${PID[@]};
do
    echo "loop pid= $var"
    kill -9 $var
done

echo "开始部署路由。。。。"

sh /root/work/route/route-startup.sh

echo "部署路由成功！"

#=======================================
# 部署server
cp /root/work/netty-action/cim-server/target/cim-server-1.0.0-SNAPSHOT.jar /root/work/server/

appname="cim-server" ;
PID=$(ps -ef | grep $appname | grep -v grep | awk '{print $2}')

# 遍历杀掉 pid
for var in ${PID[@]};
do
    echo "loop pid= $var"
    kill -9 $var
done

echo "开始部署服务1。。。。"
sh /root/work/server/server-startup.sh
echo "部署服务1成功！"


echo "开始部署服务2。。。。"
cp /root/work/netty-action/cim-server/target/cim-server-1.0.0-SNAPSHOT.jar /root/work/server2/
sh /root/work/server2/server-startup.sh
echo "部署服务2成功！"
```
```shell
nohup java -jar  /root/work/route/cim-forward-route-1.0.0-SNAPSHOT.jar  > /root/work/route/log.file 2>&1 &
```
* > /root/work/route/log.file表示把日志文件重定向
* 2>&1 0代表标准输入，1代表标准输出，2代表err输出。这里是把标准输出和err都重定向到/root/work/route/log.file
* &表示后台运行。nohup该命令可以在你退出帐户/关闭终端之后继续运行相应的进程。nohup就是不挂起的意思( nohang up)。