---
layout: post
title: Linux命令
tags:
  - linux
excerpt_separator: <!--more-->
---

因为注册了O'Reilly的会员，在查可以学习的资料时，正好补一下不那么常用，也就是我不太会的linux命令。


### 操作文件
#### rm、cp、mv
常用命令不多说

#### wildcards
正则里的规则

比如：
```
*
?
[aeg]
[0-3]
[!] not
[[:alpha:]]
```
可在 `ls`,`rm`, `cp` 等命令中组合使用

### Input, Output, redirect

| I/O Name | Abbr | File Descriptor |
|---|---|---|
|standard input| stdin | 0|
|standard output | stdout | 1|
|standard error| stderr | 2|

`>`  redirect standard output to a file. Overwrite(truncating) existing contents

`>>` redirect standard output ot a file. Appends to existing contents

`<` redirect input from a file to a command

`/dev/null`

通过fd可以任意redirect到文件或者丢掉

### 对比文件
三个命令

#### diff
```
$ diff file1 file2
2c2
< b
---
> bbb
6,7d5
< f
< g
```
`2c2`里 `2`表示file1的第二行, `c`表示 `change`，第二个`2`表示file2的第二行

action有

- `a` - add
- `c` - change
- `d` - delete

#### sdiff
side-by-side 版的diff

```
$ sdiff file1 file2
a								a
b							      |	bbb
c								c
d								d
e								e
f							      <
g							      <
```

`|` 表示不同的那行 
`<` file1 
`>` file2

#### vimdiff
vim下高亮做对比


### 搜索

#### find

```
find /root -name idip
```

#### file
display file type

```bash
$ file gitea-compose.yml
gitea-compose.yml: ASCII text

$ file p3.csv
p3.csv: UTF-8 Unicode text, with CRLF line terminators
```

#### locate
比 `find`更高效
locate是用index的，所以会更快，但是不是real time的，find是real time

#### strings
筛选出human readable的文字

#### cut
string delimiter
太复杂了不用

#### less
`more`的升级版
> less is more

### 文件传输
#### scp

#### sftp

```
sftp foobar@192.168.1.101
```

居然和ssh一样，但是可以轻松地从本地传文件，使用 `put`, `get`

### 进程，job管理

#### ps
- `-e` everything
- `-f` format
- `-h` tree
- `--forest` 带线的树状
- `-u` 带上用户，如 `ps -u foobar`
- `-p` 带pid， 如 `ps -p 1`

#### jobs
显示jobs

#### fg
正在后台进行的job挪到前台

#### bg
suspend的job放到后台运行

#### Ctrl+Z
目前前台运行的程序suspend

#### kill
`kill -l` 显示可用的signal

`kill 111` 表示正常结束pid=111的进程，默认signal是15, Term(inate)，`SIGTERM`

`kill -9 111` 表示强制结束，signal为9，是`SIGKILL`

具体可见：[Link](https://www.gnu.org/software/libc/manual/html_node/Termination-Signals.html)

### scheduling repeated jobs
#### crontab
- `crontab -l` List
- `crontab -e` edit
- `crontab -r` delete all

### 切换用户
#### su
switch user
`su` 啥都不带，默认要切换到 superuser
`su - ` 环境变量

#### whoami

#### sudo
Super User Do

`sudo su - ` 带着自己的环境变量去root ~

### history
- `history` 过去的历史输入
- `!!` 上次输入的直接执行
- `!N` N 表示第N个历史输入
- `!string` string表示某个command开始的单词
- `!:N` N 表示第N个argument
- `!^` 第一个argument
- `!$` 最后一个argument

### history search
`ctrl+r` 搜索历史记录，输入keyword，如果不是想要的，再次 `ctrl+r`可以选择下一个

`ctrl+g` cancel search

<!--more-->
### install software
#### yum - centos, red hat 等用
- `yum search`
- `yum info`
- `yum install package` `yum install -y package` 自动帮你点yes
- `yum remove `
#### rpm
install rpm file directly
- `rpm -qa` list all package (`q` query)
- `rpm -qf` 这个文件属于什么package
- `rpm -ql package` list package's files
- `rpm -ivh xxx.rpm` install (`i` install `v` verbose `h` pring hash for progress)
- `rpm -e package` erase
#### deb - ubuntu等用
`apt`
- `apt-cache search string`
- `apt-get install [-y] package`
- `apt-get remove package` remove package, leaving configuration
- `apt-get purge package` remove package, deleting configuration
#### dpkg
install dep file directly
- `dpkg -l`
- `dpkg -i xx.deb`
- `dpkg -S xx.file` check xx.file in which package

### Reference
- [Learn Linux in 5 Days and Level Up Your Career - O'Reilly](https://learning.oreilly.com/videos/learn-linux-in/9781789802610)