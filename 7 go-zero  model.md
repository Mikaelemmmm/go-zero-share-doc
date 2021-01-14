go-zero中最大特色就是model层，防缓存击穿，基本把db数据都是加载到了redis中，把redis当从库在用，真的很牛设计思路，可以去看model层源码，另外有的用户觉得不习惯 ，那你就可以放弃这里改造成gorm等，但是个人感觉这个是很牛逼的特点，还是喜欢这一点，还有我是喜欢手写sql党，不太喜欢orm，哈哈。



go-zero可以生成model，每次去在命令行敲很麻烦，我就给写成了一个shell，放在了build/mysql文件夹下

![1610624436911](/Users/seven/Desktop/go-zero文章/images/七/1610624436911.jpg)

```shell
#!/usr/bin/env bash

#生成的表名
tables=$2
#表生成的genmodel目录
modeldir=./genmodel

# 数据库配置
host=127.0.0.1
port=3306
dbname=$1
username=root
passwd=root

goctl model mysql datasource -url="${username}:${passwd}@tcp(${host}:${port})/${dbname}" -table="${tables}"  -dir="${modeldir}" -cache=true
```



这样每次直接输入db 、table就可以生成到build/mysql/genmodel中了，然后可以自己稍微修改在粘贴到服务的model中

```shell
 $ sh genModel.sh dbname username
```

