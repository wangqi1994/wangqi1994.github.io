---
title: "HDFS写流程"
catalog: true
toc_nav_num: true
date: 2020-05-11 12:00:00
subtitle: "拓展"
article_type: 0 # 0 原创，1 转载
tags:
- [Hadoop]
- [HDFS]
categories:
- [Hadoop]
- [HDFS]
updateDate: 2020-05-11 12:00:00
top: 0  #置顶
---

**HDFS写流程**
```bash
[hadoop@hadoop001 ~]$ echo 'ruozedata' >1.log
[hadoop@hadoop001 ~]$ hdfs dfs -put 1.log  /
20/05/12 09:40:38 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform...\ 
using builtin-java classes where applicable
[hadoop@hadoop001 ~]$ hdfs dfs -ls /
20/05/12 09:40:51 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform...\ 
using builtin-java classes where applicable
Found 4 items
-rw-r--r--   1 hadoop supergroup         10 2020-05-12 09:40 /1.log
drwx------   - hadoop supergroup          0 2020-05-11 16:19 /tmp
drwxr-xr-x   - hadoop supergroup          0 2020-05-11 15:35 /usr
drwxr-xr-x   - hadoop supergroup          0 2020-05-11 16:57 /wordcount2
```
![HDFS写流程图](https://i.loli.net/2020/06/12/uklR3BGMaJiqV6x.png)

写入操作对用户操作是无感知的
流程：
1.HDFS Client（client外框是操作系统，操作机器）调用FileSystem.create(filePath)方法，去和NN进行【RPC】通信！
NN 会去check这个路径的文件是否已经存在，是否有权限能够创建这个文件！
假如都ok，就去创建一个新的文件，但是这时还没写数据，是不关联任何的block。
nn根据上传的文件大小，根据块大小+副本数参数，
计算要上传多少块和块存储在DN的位置。最终将这些信息返回给客户端，
作为【FSDataOutputStream】对象

2.Client调用【FSDataOutputStream】对象的write方法，
将第一个块的第一个副本数写第一个DN节点，
写完去第二个副本写DN2；写完去第三个副本写DN3。
当第三个副本写完，就返回一个ack packet确定包给DN2节点，
当DN2节点收到这个ack packet确定包加上自己也是写完了，就返回一个ack packet确定包给
第一个DN1节点，当DN1节点收到这个ack packet确定包加上自己也是写完了，
将ack packet确定包返回给【FSDataOutputStream】对象，就标识第一个块的三个副本写完。，
其他块以此类推！

3.当所有的块全部写完， client调用 【FSDataOutputStream】对象的close方法，
关闭输出流。关闭输出流之后刷缓存区的数据（可不做）
再次调用FileSystem.complete方法，告诉nn文件写成功！

伪分布式 1台dn  副本数参数必然是设置1
dn挂了，肯定不能写入


生产分布式 3台dn  副本数参数必然是设置3
dn挂了，肯定不能写入


生产分布式 >3台dn  副本数参数必然是设置3
dn挂了，能写入


存活的alive dn节点数>=副本数 就能写成功

建议副本数为3，多余数据无太大用