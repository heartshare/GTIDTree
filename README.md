# GTIDTree
GTID技术研究

获取mysql实例的uuid

![](https://i.imgur.com/YQK2G5a.png)

<pre>
GTID
          GTID(Global Transaction ID)是对于一个已经提交事务的编号，并且是全局唯一的编号。

          在源（主）服务器上提交的每个事务，将创建和其相关联的唯一标识符的全局事物标识符。此标识符
      不但是唯一的，而且在所有的复制从库中都是唯一的。所有事物和所有GTID之间都有一对一的映射关系

          GTID实际上由UUID + TID组成。UUID是Mysql实例的唯一标识，保存在mysql数据库目录下的
      auto.cnf文件里。TID代表了该实例上已经提交的事务数量，并且随着事务提交单调递增。

      例如GTID:
          3E11FA47-71CA-11E1-9E33-C80AA9429562:23

      在同一个集群内，每个MySQL实例的server_uuid必须唯一，否则同步时，会造成IO线程不停的中断，
      重连。在通过备份恢复数据时，一定要将var目录中的auto.cnf删掉，让MySQL启动时自己生成uuid。

      如果事务是通过SQL线程回放relay-log时产生，那么GTID就直接使用binlog里的了。在MySQL 5.6中
      不用担心binlog里没有GTID，因为如果从库开启了GTID模式，主库也必须开启，否则IO线程在建立连
      接的时候就中断了。5.6的GTID对MySQL的集群环境要求是非常严格的，要么主从全部开启GTID模式，要
      么全部关闭GTID模式 
</pre>

<pre>
GTID的工作原理
          1）master更新数据时，会在事务前产生GTID，一同记录到binlog日志中。
          2）slave端的i/o线程将变更的binlog写入到本地的relay log中。
          3）sql线程从relay log中获取GTID,然后对比slave端的binlog是否存在记录。
          5）如果有记录，说明事务已经执行完毕，slave会忽略。
          6）如果没有记录，slave就会从relay log中执行该GTID的事务，并记录到binlog。
</pre>

<pre>
GTID相比传统复制的优势,GTID找点儿：
           
         在未开启GTID模式的情况下，从库用（File_name和File_pos）二元组标识执行到的位置。
      START SLAVE时，从库会先向主库发送一个BINLOG_DUMP命令，在BINLOG_DUMP命令中指定
      File_name和File_pos，主库就从这个位置开始发送binlog。

         在开启GTID模式的情况下，如果指定MASTER_AUTO_POSITION=1。START SLAVE时，从库会计
      算Retrieved_Gtid_Set和Executed_Gtid_Set的并集（通过SHOW SLAVE STATUS可以查看），然
      后把这个GTID并集发送给主库。主库会使用从库请求的GTID集合和自己的gtid_executed比较，把从库
      GTID集合里缺失的事务全都发送给从库。如果从库缺失的GTID，已经被主库pruge了呢？从库报1236
      错误，IO线程中断
</pre>