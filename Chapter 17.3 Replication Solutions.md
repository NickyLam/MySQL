## 17.3 复制方案    
&emsp;&emsp;复制可以因为很多不同的目的用在很多不同的环境中。本章节会针对特殊的方案类型来提供大概的提醒和建议。   

&emsp;&emsp;对于想了解在备份环境中使用复制的信息，包括对安装、备份和备份的文件的说明，参考[第 17.3.1 节 “Using Replication for Backups”]()。   

&emsp;&emsp;对于在master和slave上使用不同存储引擎的建议和提示，参考[第 17.3.3 节 “Using Replication with Different Master and Slave Storage Engines”]()。        

&emsp;&emsp;使用复制作为向外扩展的方法，需要对使用这个方法的应用的逻辑和运作进行修改。参考[第 17.3.4 节 “Using Replication for Scale-Out”]()。        

&emsp;&emsp;出于性能和数据分布的需要，你可能想把不同的数据库复制到不同的slave上。参考[第 17.3.5 节 “REplicating Different Databases to Different Slaves”]()。        

&emsp;&emsp;随着复制slave数量增多，master的负载会上升，导致性能的下降（因为需要复制二进制日志到每个slave）。想了解改善复制性能的提示，包括使用一个单独的二级服务器作为复制master，参考[第 17.3.6 节 “Improving Replication Performance”]()。         

&emsp;&emsp;如果想了解切换master，或者把slave转换为master来作为失效备援的一种紧急方法的指导，参考[第 17.3.7 节 “Switching Masters During Failover”]()。      

&emsp;&emsp;为了包括你的复制通信，你需要加密你的通讯渠道。想要一步步的指引，参考[第 17.3.8 节 “Setting Up Replication to Use Secure Connections”]()。        

<br />

### 17.3.1 使用复制来备份        
&emsp;&emsp;要把复制作为一种备份方案，要先把master的数据复制到slave上，然后备份数据slave。因为slave无论被暂停或者关闭都不会影响master的运行操作，所以你可以创建“活”数据的高效快照，这有可能要求master关闭。       

&emsp;&emsp;如何备份数据库，取决于数据库的大小和你是只备份数据还是备份数据和复制slave的状态，以便在事例失败的时候重建slave。因此有两种选择：       
- 如果你的数据库规模不是很大，而且你使用复制来备份master上的数据，用[mysqldump]()比较合适。参考[第 17.3.1.1 节 “Backing Up a Slave Using mysqldump”]()。            
- 对于大型数据库，[mysqldump]()就会不太实用和高效，那么你可以用原始数据文件来代替。使用原始数据文件，意味着你要备份二进制和中继日志，这些日志能够让你在slave失败时重建slave。参考[第 17.3.1.2 节 “Backing Up Raw Data From a Slave”]()。       

&emsp;&emsp;另一个对master和slave都适用的备份方案就是，把服务器设置为只读状态。这种备份用在只读的服务器上，然后把服务器改回平常的读写操作状态。参考[第 17.3.1.3 节 “Backing Up a Master or Slave by Making It Read Only”]()。        


#### 17.3.1.1 用mysqldump来备份slave    
&emsp;&emsp;适用[mysqldump]()来创建数据库副本，能够让你用一个格式来捕捉数据库的所有数据，这样能够把信息导入到另一个MySQL服务器的实例中（参考[第 4.5.4 节 “mysqldump--A Database Backup Program”]()）。因为信息的格式是SQL语句，所以当你需要紧急情况下访问数据的时候，文件能够轻易地被分配和应用到运行中的服务器上。然而，如果你的数据集太大的话，[mysqldump]()可能不太实用。        
当使用[mysqldump]()的时候，在备份开始前先把slave的复制中止，确保复制包含一协调集数据：    

1. 停止slave处理请求。你可以用mysqladmin完全中止slave上的复制过程：   
```shell
shell> mysqladmin stop-slave
```
&emsp;&emsp;或者，你可以只中止slave的SQL线程来暂停事件执行：    
```shell
shell> mysql -e 'STOP SLAVE SQL_THREAD；'    
```
&emsp;&emsp;这使得slave可以继续从master的二进制日志中接收数据更改事件，并把它们用I/O线程存储在中继日志里，但同时阻止slave执行这些事件和改变它的数据。在复制繁忙的环境中，在备份时允许I/O线程继续运行，可以加快重启slave的SQL线程的恢复过程。     

2. 使用[mysqldump]()来备份数据库。你可以选择全部或者部分数据来备份。比如说，备份全部数据库：
```shell
shell> mysqldump --all-databases > fulldb.dump
```
3. 一旦备份完成，重启slave的操作：
```shell
shell> mysqladmin start-slave
```

&emsp;&emsp;在先前的例子中，你可能想添加登陆认证信息（用户名，密码）到命令中，并把这个过程捆绑到一个每天自动运行的脚本中。     
&emsp;&emsp;如果你使用这个方法，确保你监视slave的复制过程来保证备份所用的时间没有影响到slave维持master事件的能力。参考[第 17.1.4.1 节 “Checking Replicaiton Status”]()。如果slave无法维持，你可能需要添加另一个slave和分配备份过程。想知道如何配置这种场景的例子，参考[第 17.3.5 节 “Replicating Different Databases to Different Slaves”]()。       

#### 17.3.1.2 备份slave的原始数据文件
&emsp;&emsp;为了保证你复制的数据的完整性，在slave服务器关闭的时候才在你的MySQL复制slave上备份原始数据文件。如果MySQL服务器仍在运行，后再任务可能仍在更新数据库文件，尤其是那些涉及到带有[InnoDB]()等后台过程的存储引擎。有了[InnoDB]()，在故障修复的时候这些问题应该被解决了。但是由于slave服务器在备份过程中被关闭，不会影响master的执行，所以利用这个能力还是有优势的。    
&emsp;&emsp;如果要关闭slave MySQL服务器和备份文件，则要：
1. 关闭slave MySQL服务器：
```shell
shell> mysqladmin shutdown
```
2. 复制数据文件。你可以使用任何使用的复制或者压缩工具，包括[cp]()、[tar]()或者[WinZip]()。举个例子，假设数据牡蛎在当前目录下，你可以用下面命令来压缩整个目录：
```shell
shell> tar cf /tmp/dbbackup.tar ./data
```
3. 重启MySQL服务器。在Unix下：
```shell
shell> mysqld_safe &
```         
&emsp;&emsp;在Windows下：

```shell
C:\> "C:\Program Files\MySQL\MySQL Server 5.6\bin\mysqld"
```
&emsp;&emsp;一般来说，你需要备份slave MySQL服务器的整个数据目录。如果你想作为slave来恢复数据和进行操作（比如说，在slave失败事例中），那么除了slave的数据外，你还应该备份slave状态文件、master信息和中继日志信息资源库和中继日志文件。在恢复slave的数据后，这些文件需要用来恢复复制过程。     
&emsp;&emsp;如果你丢失了中继日志，但是还有[relay-log.info]()文件，你可以检查一下来确定在master二进制日志中SQL线程执行到什么程度。然后你可以用[CHANGE MASTER TO]()带上[MASTER_LOG_FILE]()和[MASTER_LOG_POS]()选项来让slave重读那个点的二进制日志。这需要二进制日志仍然在master服务器上。       
&emsp;&emsp;如果你的slave在复制[LOAD DATA INFILE]()语句，你还应该备份在slave为了这个目的所使用的所有目录上的任何[SQL_LOAD-*]()文件。slave需要这些文件来在被任何[LOAD DATA INFILE]()操作中断后恢复复制过程。目录位置是[--slave-load-tmpdir]()选项的值。如果服务器启动的时候没有这个选项，那么目录位置就是系统变量[tmpdir]()的值。 

#### 17.3.1.3 通过设置只读状态来备份master或者slave   
&emsp;&emsp;在复制设置时，通过获取全局读锁并操作全局变量[read_only]()来改变服务器的只读状态成备份状态来备份master或者slave服务器是可能的：
1. 使服务器出于只读状态，这样它只处理检索和锁更新。
2. 开始备份。
3. 把服务器设置回正常的读写状态。
> 注意    
> 本章节的指令把服务器放在一个对从服务器获取数据的备份方法如[mysqldump]()（参考[第 4.5.4 节 “mysqldump--A Database Backup Program”]()）安全的状态来备份。你不用尝试通过直接复制文件来进行二进制备份，因为服务器可能在内存中有被修改的数据缓存并且没有回滚到硬盘中。   

&emsp;&emsp;接下来的指令讲述如何对master服务器和slave服务器做到这一步。对于这里讨论到的两种场景，我么你假设你已经执行一下复制配置：   
- 一台master服务器 M1
- 一台设置M1作为master的slave服务器 S1
- 一台连接到M1的客户端C1
- 一台连接到S1的客户端C2

&emsp;&emsp;无论在哪种场景，服务器都会执行获取全局读锁并且操作[read_only]()变量的语句来备份,并且不会影响到该服务器下的任何slave。    

**场景一：备份一台只读状态的Master** 
通过执行下列语句来设置master M1为只读状态：
```SQL
mysql> FLUSH TABLES WITH READ LOCK;
mysql> SET GLOBAL read_only = ON;
```
&emsp;&emsp;当M1出于只读状态时，一下属性都是真的：    
- 由于服务器出于只读状态，由C1到M1的更新请求会被屏蔽。
- 可以从C1到M1进行查询请求。
- 在M1上备份是安全的。
- 在S1上备份是不安全的。服务器仍在运行，而且可能会处理二进制日志或者是从客户端C2发来的更新请求。

&emsp;&emsp;在M1只读时进行备份。比如说，你可以使用[mysqldump]()。  
&emsp;&emsp;当M1上的备份完成时，通过下列语句来把M1恢复到正常的操作状态：
```SQL
mysql> SET GLOBAL read_only = OFF;
mysql> UNLOCK TABLES;
```
&emsp;&emsp;尽管在M1上备份是安全的（对于备份来说），但是这不是性能最优的，因为M1的客户端的更新请求被屏蔽。   

&emsp;&emsp;这种策略用在复制配置时备份master服务器，也可以用在非复制配置时的单个服务器。   

**场景 2：备份出于只读状态的slave**         
执行下列语句来设置S1为只读状态：
```SQL
mysql> FLUSH TABLES WITH READ LOCK;
mysql> SET GLOBAL read_only = ON;
```
当S1处于只读状态的时候，以下性质是真的：   
- master M1继续运作，因此备份master是不安全的。
- S1被中止了，所以备份S1是安全的。                    

&emsp;&emsp;以上性质给一个常见的备份场景提供了基础：让一台slave进行复制一段时间不是问题，因为这不会影响整个网络，并且复制的时候系统仍然在运行。特别的，客户端让然可以在master服务器上更新数据，并且不会影响到slave的备份。     
&emsp;&emsp;当S1处于只读状态的时候进行备份。比如说，使用[mysqldump]()。       
&emsp;&emsp;当S1的备份结束后，通过执行以下语句来把S1恢复到正常的运行状态：
```SQL
mysql> SET GLOBAL read_only = OFF;
mysql> UNLOCk TABLES;
```

&emsp;&emsp;当slave恢复到正常的操作后，它会再次根据master的二进制日志来通过抓取一些明显的更新来跟master同步。        


### 17.3.2 处理slave复制过程中的突发中止状况  
&emsp;&emsp;为了让slave在服务器意外中止时也能恢复过来，slave必须在中止前可以恢复它的状态。本章节描述了在复制过程中slave意外中止的影响，以及如何配置slave使得能够最快恢复来继续复制。      
  
&emsp;&emsp;在slave意外中止后，我们先要恢复已经接受的事务的信息，并且SQL线程要恢复已经执行的事务，再重启I/O线程。对于恢复所需要的slave日志的信息，参考[第 17.2.2 节 “复制中继和状态日志”]()。恢复需要的信息一般存储在文件中，这根据slave中止时事务处理到的阶段，甚至是文件本身的崩溃，会带来一定与master失去同步的风险。在MySQL5.6，你可以用表来存储这些信息作为代替。这些表用[InnoDB]()创建，并且通过这个事务性的存储引起，这些信息都可以在重启时得到恢复。要在MySQL5.6中设定配置来把复制信息存储在表中，把[relay_log_info_repository]()和[master_info_repository]()设成[TABLE]()。服务器接着会把I/O线程恢复所需要的信息和SQL线程恢复所需要的信息分别存在[mysql.slave_master_info]()表和[mysql.slave_relay_log_info]()表中。      

&emsp;&emsp;准确地说，复制的slave如何从意外中止中恢复，受到所选择的复制的方法、slave是单线程还是多线程、[relay_log_recovery]()等变量的设置和像[MASTER_AUTO_POSITION]()等变量是否被使用等因素的影响。    
&emsp;&emsp;下面这张表展示了不同因素对单线程slave从意外中止中恢复的影响。   

**表 17.3 影响单线程复制slave恢复的因素**

|GTID|MASTER_AUTO _POSITION|relay_log _recovery|relay_log_info _repository|Crash type  |Recovery guaranteed|Relay log impact|
|:--:|:-------------------:|:-----------------:|:------------------------:|:----------:|:-----------------:|:--------------:|
|OFF |Not relevant         |1                  |TABLE                     |Any         |Yes                |Lost            |
|OFF |Not relevant         |1                  |TABLE                     |Server      |Yes                |Lost            |
|OFF |Not relevant         |1                  |Any                       |OS          |No                 |Lost            |
|OFF |Not relevant         |0                  |TABLE                     |Server      |Yes                |Remains         |
|OFF |Not relevant         |0                  |TABLE                     |OS          |No                 |Remains         |
|ON  |ON                   |1                  |Not relevant              |Not relevant|Yes                |Lost            |
|ON  |OFF                  |0                  |TABLE                     |Server      |Yes                |Remains         |
|ON  |OFF                  |0                  |Any                       |OS          |No                 |Remains         |


&emsp;&emsp;如表所示，当使用单线程slave时，以下配置是在意外中止后最容易恢复的：    
- 当使用GTID和[MASTER_AUTO_POSITION]()时，设[relay_log_recovery=1]()。有了这个配置，[relay_log_info_repository]()和其他变量的设置就不会影响到恢复了。
- 当时用到给予复制的文件位置时，设[relay_log_recovery=1]()，[relay_log_info_repository=TABLE]()。  

> **注意**        
> 在恢复过程中，中继日志是缺失的。



**表 17.4 影响多线程复制slave恢复的因素**

|GTID|sync_relay _log|MASTER_AUTO _POSITION|relay_log _recovery|relay_log_info _repository|Crash type  |Recovery guaranteed|Relay log impact|
|:--:|:-------------:|:-------------------:|:-----------------:|:------------------------:|:----------:|:-----------------:|:--------------:|
|OFF |1              |Not relevant         |1                  |TABLE                     |Any         |Yes                |Lost            |
|OFF |>1             |Not relevant         |1                  |TABLE                     |Server      |Yes                |Lost            |
|OFF |>1             |Not relevant         |1                  |Any                       |OS          |No                 |Lost            |
|OFF |1              |Not relevant         |0                  |TABLE                     |Server      |Yes                |Remains         |
|OFF |1              |Not relevant         |0                  |TABLE                     |OS          |No                 |Remains         |
|ON  |Any            |ON                   |1                  |Any                       |Any         |Yes                |Lost            |
|ON  |1              |OFF                  |0                  |TABLE                     |Server      |Yes                |Remains         |
|ON  |1              |OFF                  |0                  |Any                       |OS          |No                 |Remains         |


&emsp;&emsp;如表所示，当使用多线程slave时，以下配置是在意外中止后最容易恢复的：    
- 当使用GTID和[MASTER_AUTO_POSITION]()时，设[relay_log_recovery=1]()。有了这个配置，[relay_log_info_repository]()和其他变量的设置就不会影响到恢复了。
- 当时用到给予复制的文件位置时，设[relay_log_recovery=1]()设，[sync_relay_log=1]()，[relay_log_info_repository=TABLE]()。  

> **注意**        
> 在恢复过程中，中继日志是缺失的。

&emsp;&emsp;值得注意的是[sync_relay_log=1]()的影响很大，因为它在每次事务中都会对中继日志进行写入。尽管这个方法在意外中止时最容易恢复，最多导致一个未写的事务丢失，但是它还是有可能会大大提升存储的负载。如果没有设置[sync_relay_log=1]()，那么意外中止的影响取决于操作系统如何处理中继日志。同时要注意当[relay_log_recovery=0]()时，在下次slave从意外中止后启动时，中继日志会处理为恢复的一部分。在这个过程完成后，中继日志会被删除。         

&emsp;&emsp;如果一个多线程复制slave采用了上述复制配置中推荐的文件位置，它意外中止后可能会导致中继日志的事务不一致（事务序列的间隔）。参考[Replication and Transaction Inconsistencies]()。在MySQL 5.7.13及以后的版本中，如果中继日志恢复过程中出现这样的事务不一致，它们会被填充，并且恢复过程自动继续。在MySQL 5.7.13以前的版本中，这个过程并不会自动进行，要求设置[relay_log_recovery=0]()来启动服务器，用[START SLAVE UNTIL SQL_AFTER_MTS_GAPS]()来启动slave，从而修补事务部一致性，再设置[relay_log_recovery=1]()来重启slave。      

&emsp;&emsp;当你是用多源复制，并且设置[relay_log_recovery=1]()时，由于意外中止的缘故，在重启后，所有复制通道将会进行中继日志恢复过程。任何由多线程slave的意外中止导致的中继日志中的不一致性将会被修复。            

### 17.3.3 用不同的master和slave存储引擎进行复制    
&emsp;&emsp;对于复制过程来说，master的源表和slave上的复制表是否使用不同的引擎类型并不重要。实际上，[default_storage_engine]()和[storage_engine]()两个系统变量并不会被复制。     

&emsp;&emsp;这提供了复制过程的一系列好处，你可以在不同的复制场景利用不同的引起类型的优势。例如，在一个典型的向外扩展的场景（参考[第 17.3.4 节 用复制向外扩展]()），你想在master上用[InnoDB]()表来有利于事务的函数化，但是在slave上使用[MyISAM]()，因为数据是只读的，所以不需要事务支持。当你在数据日志记录环境中，你可能想在slave上用[Archive]()引擎。         

&emsp;&emsp;如何在master和slave上配置不同的引擎取决于你如何初始化复制过程：   
- 如果你是用[mysqldump]()在master上创建数据库快照，那么你可以编辑复制文件文本来修改用在每张表上的引擎类型。<br />  另一个[mysqldump]()的选择就是在用复制文件在slave创建数据前把不想使用的引擎类型关闭。例如，你可以在你的slave上通过添加[--skip-federated]()选项来把[FEDERATED]()引擎关闭掉。如果用于创建一个表的某种特定引擎类型不存在，那么MySQL会采用默认引擎类型，通常是[MyISAM]()（这要求关闭[NO_ENGINE_SUBSTITUION]()模式）。如果你想用这种方法关闭额外的引擎，那么你可以考虑在slave上创建特殊的二进制文件来只支持你想用的引擎类型。      
- 如果你用原始数据文件（二进制备份）来创建slave，那么你不能够改变出事的表格式。相反，你可以在slave启动后，用[ALTER TABLE]()来改变表类型。  
- 当master恰好没有表时想创建新的master/slave复制，在创建新表时避免指定特定引擎类型。        


&emsp;&emsp;如果你已经在运行一个复制过程，并且想把你已有的表格转换成另一种引擎类型，遵循以下步骤： 
1. 从运行中的复制更新过程中关闭slave：
```SQL
mysql> STOP SLAVE;
```
&emsp;&emsp;这会让你在改变引擎类型时不受干扰。

2. 对每张表执行[ALTER TABLE ... ENGINE='engine_type']()来改变。   
3. 重启slave复制过程：
```SQL
mysql> START SLAVE;
```

&emsp;&emsp;尽管[default_storage_engine]()变量不会被复制，但要注意，包括指定引擎类型的[CREATE TABLE]()和[ALTER TABLE]()变量会被正确地复制到slave上。例如，如果你有一个CSV表，并且你执行：
```SQL 
mysql> ALTER TABLE csvtable Engine='MyISAM';
```
&emsp;&emsp;上述语句会被复制到slave上，并且slave的引起类型会被改成[MyISAM]()，即使你没有事先在slave上把表类型改成CSV意外的其他类型。如果你想要保持master和slave的引擎不同，那么你在master上要小心使用[default_storage_engine]()变量来创建新表。例如，代替：
```SQL
mysql> CREATE TABLE tablea (columna int) Engine=MyISAM;
```
&emsp;&emsp;你可以用：
```SQL
mysql> SET default_storage_engine=MyISAM;
mysql> CREATE TABLE tablea (columna int);
```
&emsp;&emsp;当复制完，[default_storage_engine]()变量会被忽略，slave会用默认引擎来执行[CREATE TABLE]()语句。             

