# 将csv数据导入hive并转换为parquet table
## 为什么要用parquet table?
hive的存储有好几种不同的格式。对比见下：
    1.textfile
    textfile为默认格式
    存储方式：行存储
    磁盘开销大 数据解析开销大
    压缩的text文件 hive无法进行合并和拆分

    2.sequencefile
    二进制文件,以<key,value>的形式序列化到文件中
    存储方式：行存储
    可分割 压缩
    一般选择block压缩
    优势是文件和Hadoop api中的mapfile是相互兼容的。

    3.rcfile
    存储方式：数据按行分块 每块按照列存储
    压缩快 快速列存取
    读记录尽量涉及到的block最少
    读取需要的列只需要读取每个row group 的头部定义。
    读取全量数据的操作 性能可能比sequencefile没有明显的优势

    4.orc
    存储方式：数据按行分块 每块按照列存储
    压缩快 快速列存取
    效率比rcfile高,是rcfile的改良版本

    5.自定义格式
    用户可以通过实现inputformat和 outputformat来自定义输入输出格式。

参照网上的对比，效率大概是这样的。
![compare](https://github.com/StillMas/Distributed-test/raw/master/hive_datetype.png)
所以我们必须用parquet或者orc，不能用低效的textfile
## 第一步：导入csv
很不幸，hive只支持用textfile形式导入csv。所以我们首先得把csv弄进来。
hive存储csv需要用到SerDe形式，具体是怎么回事可以参照https://blog.csdn.net/zengmingen/article/details/52636385

    drop table if exists test;
    create table if not exists test(  
    appid5 string COMMENT '1',   
    timestr string COMMENT '2')
    partitioned by (   
    datestr string,  
    collect_time string)
    ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde' 
    WITH SERDEPROPERTIES ('separatorChar'=',') 
    STORED AS TEXTFILE;

最后的stored as是核心。建表完成后即可开始导入本地的csv文件。

    load data local inpath '/disk2/test.csv' into table test 
    partition (datestr='2018_06_08',collect_time='00_35_17');

注意建表的时候，选择了textfile，就默认所有字段全部是string。但在之后转换为parquet或者orc的时候，string可以直接变成其他数据格式，不会冲突。
如果csv文件很多，可以写个简单的python脚本批量生成命令。

    path = r'/disk2/'
    os.chdir(path)
    filelist = []
    hivecommands = []
    for root, dirname, files in os.walk(path):
        for file in files:
            if os.path.splitext(file)[-1] == '.csv':
                filepath = os.path.join(root, file)
                LDC_id = filepath.split('/')[5][-1]
                train_id = filepath.split('/')[3]
                hivecommand = 'load data local inpath \'%s\' into table test partition (datestr=\'%s\',collect_time=\'%s\');'\
                              % (filepath, LDC_id, train_id)
                filelist.append(filepath)
                hivecommands.append(hivecommand)
                train_ids.append(train_id)
                i+= 1
        if i > 100000:
            break

    with open('/home/inn/hive_commands.txt', 'wb') as f:
        f.write('\n'.join(hivecommands))
        
之后采用Hive CLI或者Beeline执行这个文件，以hive CLI为示例。别忘了开个screen让他慢慢跑。
    
    screen -S hive_import
    cd /home/inn 
    hive -f hive_commands.txt

## 第二步：将完整的textfile表转换成parquet表
如果我们的textfile表不算很大，那么转换就很简单了。


首先，由于我们的表有partition（partition是个好东西，人人都应该有），所以必须要在转换之前设置hive的动态分区。

    set hive.exec.dynamic.partition=true;
    set hive.exec.dynamic.partition.mode=nonstrict;
    set hive.exec.max.dynamic.partitions.pernode=1000;

千万注意，由于这个set一旦离开hive当前环境就失灵了，所以最好把这三句和后面的所有hive脚本写入一个txt文件，统一用hive -f运行。
之后，我们建立一个parquet表。

    drop table if exists test_parquet;
    create table if not exists test_parquet (  
    appid5 string COMMENT '1',   
    timestr string COMMENT '2')
    partitioned by (   
    datestr string,  
    collect_time string)
    ROW FORMAT SERDE  'parquet.hive.serde.ParquetHiveSerDe'
    STORED AS INPUTFORMAT  'parquet.hive.DeprecatedParquetInputFormat'
    OUTPUTFORMAT  'parquet.hive.DeprecatedParquetOutputFormat';
    
然后采用insert命令，将textfile表的内容灌进来。

    insert overwrite table test_parquet partition (datestr, collect_time) 
    select appid5, timestr, datestr, collect_time
    from test;

至此大功告成。

## 例外情况：如果textfile表过大，内存容量不支持一次性转换

这时候就采用和之前一样的方法，把表按照分区，逐块导入。我们也可以写个python脚本，批量生成导入命令。

    # 手动拉出一个datestr的分区信息
    datestrs = ['2018_06_10', '2018_05_23', '2018_05_08', '2018_04_28', '2018_04_27', '2018_05_07', '2018_04_25', '2018_04_24', '2018_04_23', '2018_05_03', '2018_06_22', '2018_05_01', '2018_04_05', '2018_04_04', '2018_04_06', '2018_04_03', '2018_04_09', '2018_04_08', '2018_06_18', '2018_04_29', '2018_06_14', '2018_06_15', '2018_06_16', '2018_05_09', '2018_04_26', '2018_06_11', '2018_06_12', '2018_06_13', '2018_04_30', '2018_06_19', '2018_05_14', '2018_05_04', '2018_05_05', '2018_05_19', '2018_05_18', '2018_05_30', '2018_05_15', '2018_05_02', '2018_05_11', '2018_05_10', '2018_05_13', '2018_05_12', '2018_04_16', '2018_04_17', '2018_04_12', '2018_04_18', '2018_04_19', '2018_06_03', '2018_06_02', '2018_06_01', '2018_06_07', '2018_06_06', '2018_06_05', '2018_06_04', '2018_06_21', '2018_06_20', '2018_06_09', '2018_06_08', '2018_06_25', '2018_06_24', '2018_06_26', '2018_06_17', '2018_06_23']
    
    # 之后就可以
    insert_commands = []
      for datestr in datestrs:
          insert_command = insert_select_part + ' where datestr = \'%s\';' \
                                                  % (datestr) 
          insert_commands.append(insert_command)
          
    # 下面的create_parquet_table是一个字符串，记录了上面从set到create table的一系列代码，这里略去。
    with open(r'/home/inn/transfer_table_parquet.txt', 'wb') as f:
        f.write(create_parquet_table + '\n'.join(insert_commands))

之后在命令行中运行

    screen -S hive_import
    cd /home/inn
    hive -f transfer_table_parquet.txt

上面的几十条命令就会逐条执行，虽然这样，mapreduce每次启动任务让总体速度变慢了，但至少克服了OOM。
