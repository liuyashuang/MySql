## Mysql数据库的优化技术
   对mysql优化是一个综合性的技术
   ### 优化方式
    1 表的设计合理化（符合 3NF）
    2 添加适当索引（index) 【四种：普通索引、主键索引、唯一索引、全文索引】
    3 分表技术（水平分割、垂直分割）
    4 读写分离【写：update/delete/add】
    5 存储过程【模块化编程，可以提高速度】
    6 对mysql配置优化【配置最大并发数 my.ini  max_connections;调整缓冲大小】
    7 mysql服务器硬件升级
    8 定时的清除不需要的数据，定时进行碎片整理

#### 1. 数据库表的设计
    1NF:要求表的列（属性）具有原子性，不能再分割，只要数据库是关系型数据库就自动满足1NF
        数据库分类
        a 关系型数据库
        b 非关系型数据库（特点：面向对象和集合）
        c NoSQL数据库（特点:面向文档）

    2NF:表中的记录是唯一的，通常设计一个主键就实现了

    3NF:表中不要有冗余数据，即表的信息能推导出来，就不应该单独设计一个字段

#### 2. Sql语句的优化
- ##### 如何在项目中，迅速的定位执行速度慢的语句（定位慢查询）
**mysql数据库的一些运行状态如何查询（运行时间/一共执行了多少次/当前连接）**
`show status`
常用的：
`show status like 'uptime'`; //查询执行多长时间</br>
<br/>`show status like 'com_select' show status like com_insert ···` //执行语句的次数</br>
<br/>`show [session|global] status like ··· `//如果不写[session|global] 默认是session 会话（当前窗口）</br>
<br/>`show status like 'connections'` //当前连接数</br>
`show status like 'slow_queries'` //查询慢查询次数</br>
- ##### 如何定位慢查询（mysql默认10秒是慢查询）
 `show variables like 'long_query_time'` //显示当前慢查询时间</br>
 <br/>`set long_query_time=1` //修改慢查询时间</br>
 <br/>`delimiter $$ `//修改命令执行结束符修改为'$$'</br>

- 如何把慢查询的sql记录到日志中
<br/>默认情况下，mysql不会记录慢查询，需要在启动mysql时候，指定记录慢查询</br>
`bin\mysqld.exe --safe-mode --slow-query-log`[mysql5.5可以在my.ini指定，在bin目录上一级执行这个命令]
- ##### 优化问题
   - **通过 explain语句可以分析，mysql如何执行你的sql语句**
   - **添加索引（主键索引、唯一索引、全文索引、普通索引）**
      - **添加**
         - **主键索引**
         <p/>将某个列设置为主键是，则该列就是主键索引。创建表后再添加,指令：`alter table 表名 add primary key(列名)`</P>
         - **普通索引**
         <p/>一般来说，普通索引的创建，是先创建表，然后再创建普通索引
         `create table 表名(···) create index 索引名 on 表名(列) `</p>
         - **全文索引**
         <p/>全文索引主要是针对文本的检索，比如文章，全文索引针对MyISAM生效</p>
         ```
         //创建
         create table articles(
             id INT UNSIGNED AUTO_INCREMENT NOT NULL PRIMARYKEY,
             title VARCHAR(200),
             body TEXT,
             FULLTEXT(title,body)
         )engine=myisam charset utf8;
         ```
         </br>全文索引错误用法：不会使用到全文索引</br>
         `select * from articles where body like '%mysql%'`；
         <br/>正确用法</br>
         `select * from articles where match(title,body) against('database')`;
         <br/>**说明：**
         <br/> 1）在mysql中 fulltext索引只针对 myisam生效
         <br/> 2）针对英文生效->sphinx(coreseek) 技术处理中文
         <br/> 3）使用方法是 match(字段名) against('关键字')
         <br/> 4）全文索引有一个停止词（对常用的一些词和字符就不会创建）
         - **唯一索引**
         <p/>当表的某一列被指定为unique约束时，这列就是一个唯一索引</p>
         ```
         //创建
         create table ddd(
             id int primary key auto_increment,
             name varchar(32) unique
         );
         //或者在创建好表之后添加， create unique index 索引名 on 表名(列名)
         ```
         <p/>unique字段可以为NULL，并且可以多个，但是只能有一个空子串('')</p>
      - **查询**
         - **desc 表名【缺点是不能显示索引名】**
         - **show index(es) from 表名\G**
         - **show keys from 表名**

      - **删除**</br>
      `alter table 表名 drop index 索引名`</br>
      **删除主键索引**
      `alter table 表名 drop primary key`

      - **修改**</br>
      先删除再创建</br></br>

      - **索引使用的注意事项**</br>
      占用磁盘空间</br>
      对 update delete insert 语句的效率影响</br>

      **在那些列上适合添加索引**</br>
      较频繁的作为查询条件字段应该创建索引</br>
      `select * from emp where empno = 1`</br>
      唯一性太差的字段不适合单独创建索引，即使频繁作为查询条件
      `select * from emp where sex='男'`</br>
      更新非常频繁的字段不适合创建索引</br>

      **一定要创建索引的几个条件**</br>
      肯定在where条件中经常使用</br>
      字段的内容不是唯一的几个值</br>
      字段内容不是频繁变化的</br>

      - **使用索引的注意事项**</br>
      对于创建的多列索引，只要查询条件使用了最左边的列，索引一般就会被使用</br>
      对于使用like的查询，查询如果是'%aaa'不会使用到索引，'aaa%'会使用到索引</br>
      如何条件中有or，即使其中有条件带索引也不会使用，就是要求所有字段都必须建立索引</br>
      如果列类型是字符串，那一定要在条件中将数据使用引号引用起来，否则不使用索引</br>
      如果mysql估计使用全表扫描要比使用索引快，则不使用索引</br>
      **查看索引使用情况**
      `show status like 'Handler_read%'`</br>
      handler_read_key :这个值越高越好，越高表示使用索引查询到的次数
      handler_read_rnd_next :这个值越高，说明查询低效</br>

      - **Sql语句小技巧**</br>
          1. 在使用 gruop by 分组查询时，默认分组后，还会排序，可能会降低速度,在后面增加一个order by null，就不会排序了</br>
          2. 有些情况下，使用连接来代替子查询，因为使用join,不需要在内存中创建临时表</br>

#### 3. mysql存储引擎的选择
    MYISAM 存储：如果对表的事务要求不高，同时是以查询和添加为主的，我们考虑使用myisam存储引擎
    INNODB 存储：对事务要求高，保存的数据都是重要数据，
    Memory 存储：数据变化频繁，不需要入库，同时又频繁的查询和修改，
    如果数据库的存储引擎是myisam，一定要定时进行碎片整理
    指令：optimize table 表名

#### 4. MySQL数据库备份
##### 1).手动备份数据库
###### <font color="#dd0000">（cmd 控制台）</font></br>
没有表名备份整个数据库</br>
mysqldump -u root -proot 数据库的名字 >文件路径</br>
备份数据库中的某张表</br>
mysqldump -u root -proot 数据库名 表名 > 文件路径</br>
使用备份文件恢复我们的数据</br>
source 备份的数据路径##### <font color="#dd0000">（mysql控制台）</font></br>

##### 2).使用定时器自动完成
把备份数据库的指令，写入到 bat文件，然后通过任务管理器去定时调用 bat文件</br>
.bat 文件内容：</br>
`bin目录的路径\bin\mysqldump -u root -proot 数据库名 表名 > 目的路径`</br>
然后可以在电脑控制面板里的计划任务进行配置</br></br>
或者编写程序来执行</br>
通过代码来调用数据库备份的命令，通过.bat文件来调用代码文件</br>

#### 5.表的分割技术
##### 1)水平分割
  按记录进行分割，不同的记录可以分开保存，每个子表的列数相同</br>
##### 2)垂直分割
  按列进行分割，即把一条记录分开多个地方保存，每个子表的行数相同</br>

#### 6.数据库参数优化的配置(配置my.ini文件)
    最大连接数：max_connections
    查询缓存的大小：query_cache_size

#### 7.增量备份
  定义：mysql数据库会以二进制的形式，自动把用户对mysql数据的操作，记录到文件</br>
  当用户希望恢复的时候可以使用备份文件，进行恢复</br>
  增量备份会记录（dml语句，创建数据库，创建表的语句，不会记录select）</br>
  记录内容：操作语句本身、操作时间、position</br>
  步骤：</br>
  1) 配置 my.ini 文件或者my.cof</br>
      log-bin='备份文件要存放的路径'</br>
  2) 重启mysql 得到文件</br>
      .index  索引文件，有哪些增量备份文件</br>
      mylog.00001 存放用户对数据库操作的文件</br>
      可以通过mysqlbinlog.exe 来查看对数据库操作的文件</br>
  3) 可以通过操作时间和position来恢复</br>
      `mysqlbinlog --stop-datetime=" " 操作文件的路径 | mysql -uroot -p`</br>
      `mysqlbinlog --start-datetime=" " 操作文件的路径 | mysql -uroot -p`</br>
      `mysqlbinlog --stop-position=" " 操作文件的路径 | mysql -uroot -p`</br>
      `mysqlbinlog --start-datetime=" " 操作文件的路径 | mysql -uroot -p`</br>
      stop:从开始时间或位置，到输入的时间、位置</br>
      start：从输入的时间、位置开始 一直到现在</br>
