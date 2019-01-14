#### grep 命令
1. 在文件中查找某个字符串
grep "hello" hello.txt
2. 统计匹配模式的行数
grep -c "love" hello.txt
3. 在正则表达式下搜索文档
grep -E "^h" hello.txt
4. 搜索多个文件并查找匹配文本在哪些文件中
grep -l "text" file1 file2 file3...
5. 在多级目录中对文本进行递归搜索（-r 递归，-n 输出行号）
grep "hello" /home/flink/maskwang -r -n
6. grep静默输出（用于测试）
grep -q "test" filename

#### sed命令
1. 替换文本中的字符串（注意是输出每一行处理后的，原来的文档并没有发生变化）
sed 's/hello/hellos/' hello.txt
2. 直接编辑文件选项-i，会匹配file文件中每一行的第一个hello替换为hellos(原来的文档会发生变化)
 sed -i 's/hello/hellos/' hello.txt
3. 使用后缀 /g 标记会替换每一行中的所有匹配
sed 's/hello/hellos/g' file 
4. 当需要从第N处（每一行）匹配开始替换时，可以使用 /Ng
sed 's/hello/hellos/2g' file 
5. 删除空白行
sed -i '/^$/'d hello.txt

#### awk命令（它是一门语言）
awk 'BEGIN{ commands } pattern{ commands } END{ commands }'
