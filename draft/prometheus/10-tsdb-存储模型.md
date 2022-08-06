## 概念

### Series

每一个series都是由 metric 和 label的所有可能键值对唯一组成的 比如一个metric 是  go_goroutine, 有两个标签 instance， name

那么这个metric的所有可能的series可能是如下

go_goroutine{instance="172.9.9.9",name="test1"}

go_goroutine{instance="172.8.8.89",name="test2"}

go_goroutine{instance="172.8.8.89",name="test1"}

这个概念先放在这里，这个在用户实际使用的时候可能接触的比较少，和prometheus时序数据库的实现有关联


### Samples

对应的是一个具体的值，简单理解就是Series加上时间和数字，prometheus每隔几秒采集到的目标的监控值就是samples。

 ### Block

 一个逻辑上的概念，时序数据库的存储是按照时间序列为主线，每个时间段的数据存储在一起（比如两个小时），这就是block。对应到prometheus的实体就是 data 目录下那些id为名称的目录。每个block文件夹中有meta.json 
 有 index文件 有chunks文件夹（这里边放的就是数据存储的文件，每个文件最大512M），还有tomstones（标记数据删除的，因为prometheus只会在数据超过存储最大时间时才会真正删除文件）

 ### Chunk 

 数据保存的最小单位，保存了一组samples