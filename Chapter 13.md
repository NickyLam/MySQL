# Chapter 13 
## 13.4 Replication Statements  
我们可以通过SQL接口使用本章节描述的声明来控制复制。一组声明控制master服务器，另一组控制slave服务器。
### 13.4.1 SQL Statements for Controlling Master Servers    
本章节讨论管理msater复制服务器的声明。[Section 13.4.2 “SQL Statements fro Controlling Slave Servers”]() 讨论管理slave服务器的声明。        
除了这里描述的声明以外，下列 SHOW 声明也在复制过程中被用在master服务器上。关于这些声明，可以参考[Section 13.7.5 “SHOW Syntax”]()。
- [SHOW BINARY LOGS]()  
- [SHOW BINLOG EVENTS]()
- [SHOW MASTER STATUS]()    
- [SHOW SLAVE HOSTS]()  

#### 13.4.1.1 PURGE BINARY LOGS Syntax  
```SQL
PURGE { BINARY | MASTER } LOGS
    { TO 'log_name' | BEFORE datetime_expr }
```
二进制日志是包含MySQL服务器数据修改过程信息的一些列文件。日志由一些列的二进制日志和一个索引文件（参考 [Section 5.4.4, "The Binary Log"]()）组成的。      
[PURGE BINARY LOGS]() 声明删除在指定日志名或者日期前日志索引文件列出的所有二进制日志文件。[BINARY]() 和 [MASTER]() 是同义词。被删除的日志文件也会从索引文件中记录的列表中移除，这样，指定的日志文件就会成为列表里的首位。    
如果服务器没有通过 [--login-bin]() 选项来启动二进制登录的话，这些声明是没用的。  
例如： 
```SQL
PURGE BINARY LOGS TO 'mysql-bin.010';
PURGE BINARY LOGS BEFORE '2008-04-02 22:46:26';
```
[BEFORE]() 变量后的 [datetime_expr]() 参数应该是一个DATETIME值（一个[“YYYY-MM-DD hh:mm:ss”]() 格式的值）。   
在slave服务器在复制的时候，运行这个声明是安全的。你不需要停止它们。如果你有个运行中的slave恰好在读取你尝试删除的日志文件中的一个，那么这个声明就无效了。在MySQL5.6.12及后面的版本中，这会引起一个警告。（Bug #13727933）然而，如果一个服务器没有连接上，而你又碰巧要清楚一个它还没有读取的日志文件，那么slave在恢复连接后则无法进行复制。    
为了能够安全地清楚二进制日志文件，请遵循一下流程：
1. 在每台slave服务器上，用 [SHOW SLAVE STATUS]() 来检查是否在读取日志文件。
2. 用 [SHOW BINARY LOGS]() 获取master服务器上的一系列二进制日志文件。
3. 设定所有slave服务器中最早的日志文件作为目标文件。如果所有slave服务器都是最新的，那么最后列表上最后一个日志文件就是目标文件。
4. 备份所有你准备要删除的日志文件。（这个步骤是可选的，但是一直都推荐执行。）
5. 清楚除了目标文件外所有的日志文件。    
你也可以通过设置系统变量[expire_logs_days]() 来在给定的天数后自动终止二进制日志文件（参考[Section 5.1.5, “Server System Variables”]()）。如果你正在复制，你应该把这个变量设置不小于你的slave可能比master延迟的最大天数。  
当[.index]()文件所列出的二进制文件被其他方法（如在Linux中使用[rm]()）删除时，[PURGE BINARY LOGS TO]() 和[PURGE BINARY LOGS BEFORE]() 都会报错。