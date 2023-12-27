## 使用文件输入输出符进行复制
read file`< file`
write to file `> file`
> $ cat < hello.txt > hello2.txt

write some thing to append in file. `>> file`

## 管道使用
Example:
> $ curl --head --silent google.com | grep -i content-length | cut -d ' ' -f2