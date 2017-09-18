# 17章 复制    


#### 17.1.1.5 使用mysqldump创建数据快照 
一种在已有master数据库创建数据快照的方法就是用[mysqldump]()工具来对你想复制的数据库创建转储。一旦数据转储完成了，你可以在复制开始前把数据引入到salve中。    
这里展示的例子把所有的数据库转储到一个叫作[dbdump.db]()的文件中，其中的[--master-data]()选项会自动把[CHANGE MASTER TO]()声明加入到slave中来开始复制过程：
```sh
shell> mysqldump --all-databases --master-data > dbdump.db
```
如果你没有使用[--master-data]()，那么你有必要再运行[mysqldump]()前在一个隔离的对话中手动地锁上所有表（用[FLUSH TABLES WITH READ LOCK]()），再推出或者在另一个对话中用UNLOCK TABLES来解锁。你同时必须要用[SHOW MASTER STATUS]()来获取匹配快照的二进制日志位置信息，并用它来执行恰当的[CHANGE MASTER TO]()来启动slave。 
在选择转储所包括的数据库时，记得把每个slave中你不想被复制的数据库给过滤掉。    
导入数据时，可以把dump文件复制到slave中，或者远程连接slave时访问master上的文件。  

#### 17.1.1.6 用原始数据文件创建数据快照 
如果你的数据库很大，那么复制原始数据文件会比使用[mysqldump]()来导入文件到每个slave更加高效。这项技术节省了[INSERT]()重用时更新索引的成本。 


#### 17.1.1.9 在已有复制环境添加Slave    
要对已有的复制配置添加新的slave，可以不用关闭master就实现。换句话说，除非你给心的slave配置一个不同的[server-id]()，否则你可以通过复制已有的slave来创建新的slave。    
要复制一个已有的slave：  
1. 关闭已有的slave：  
```shell
shell> mysqladmin shutdown
```
2. 把已有slave的数据目录复制到新的slave上。可以通过用 [tar]() 或者 [WinZip]() 来创建存档，或者用 [cp]() 和 [rsync]() 等工具来直接复制。确保也复制日志文件和中转日志文件。 


