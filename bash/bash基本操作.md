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