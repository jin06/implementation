 文件在次盘中的组织结构

    ./data
    ├── 01BKGV7JBM69T2G1BGBGM6KB12
    │   └── meta.json
    ├── 01BKGTZQ1SYQJTR4PB43C8PD98
    │   ├── chunks
    │   │   └── 000001
    │   ├── tombstones
    │   ├── index
    │   └── meta.json
    ├── 01BKGTZQ1HHWHV8FBJXW1Y3W0K
    │   └── meta.json
    ├── 01BKGV7JC0RY8A6MACW02A2PJD
    │   ├── chunks
    │   │   └── 000001
    │   ├── tombstones
    │   ├── index
    │   └── meta.json
    ├── chunks_head
    │   └── 000001
    └── wal
        ├── 000000002
        └── checkpoint.00000001
            └── 00000000


 每隔两小时生成一个目录，目录下时存储的数据，生成的目录下有一个文件夹chunks，chunks目录中的文件每个最大为512MB。这个chunks下保存了这两个小时内所有的样本数据。元数据存放在meta.json。 数据删除，数据不会立即删除，而是将删除的记录放在tomstones文件中。

 当前的block存放在内存中，并且不是完全持久化的。依赖于wal作故障恢复，wal日志存放在wal目录下，每个文件128MB。wal文件没有被压缩。