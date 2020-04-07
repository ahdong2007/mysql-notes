###### MySQL binlog中的概念比较多，特别是5.7引入group commit后,很容易让人疑惑。下面是整理的一下概念。

##### 1.at position

用mysqlbinlog解析出来的binlog很容易看到如下内容:

    # at 377
    # at 455
    #200401 10:58:05 server id 2731754674  end_log_pos 455  Table_map: `coredb`.`web_trd_log` mapped to number 130869
    #200401 10:58:05 server id 2731754674  end_log_pos 708  Write_rows: table id 130869 flags: STMT_END_F
    
    '/*!*/;
    ### INSERT INTO `coredb`.`web_trd_log`
    ### SET
    ###   @1='6133c68220f74bb794585310705ff57e' /* VARSTRING(150) meta=150 nullable=0 is_null=0 */
    ###   @2='a83cbc5b3262483f9a84d331482bf56b' /* VARSTRING(150) meta=150 nullable=0 is_null=0 */

这里的at 455指的是binlog的第455字节,因而在默认max_binlog_size=1G的配置下,binlog几乎都是# at 4开头，以# at 1073744392左右结尾(1G大小)。

##### 2.gtid

全局唯一事务号,两个不同事务的gtid必定不相同,MySQL官方版本binlog中形式这样:

```
SET @@SESSION.GTID_NEXT= '02a6e9e4-d92b-11e9-85c5-6c0b84d5e9be:27247752'/*!*/;
```

MariaDB的gtid形式就有些不一样了,binlog中会这样记录:

```
/*!100001 SET @@session.gtid_domain_id=0*//*!*/; 
/*!100001 SET @@session.server_id=82219*//*!*/; 
/*!100001 SET @@session.gtid_seq$_no=164690165*//*!*/; 
```

##### 3.sequence_number

```
 #at 587
 #200331 22:39:03 server id 3155238109  end_log_pos 648  GTID    last_committed=1        sequence_number=2
```

这个在每个binlog产生时从1开始然后递增,每增加一个事务则sequencenumber就加1,为什么有了gtid还需要再加个sequencenumber来标识事务呢？下面会解释。

##### 4.lastcommitted

```
...... 
xxxxxxxxxxxx   GTID    last_committed=3        sequence_number=8   
xxxxxxxxxxxx   GTID    last_committed=3        sequence_number=9   
xxxxxxxxxxxx   GTID    last_committed=9        sequence_number=10   
...... 
xxxxxxxxxxxx   GTID    last_committed=9        sequence_number=24   
xxxxxxxxxxxx   GTID    last_committed=24        sequence_number=25   
...... 
```

这代表sequencenumber=10到sequencenumber=24的事务在同一个组里(因为lastcommitted都相同,都是9)
lastcommitted的引入和group commit有关,这也是MySQL基于组提交(logic clock)的并行复制方式即使在gtid关闭情形下也能生效的原因。

##### 5.xid

根据官方文档说明，这是用来标识xa事务的

`Binlog::XID_EVENT:
Transaction ID for 2PC, written whenever a COMMIT is expected.`
