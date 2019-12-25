## DexHeader

* Android环境：6.0

* [完整代码地址](https://www.androidos.net.cn/android/6.0.1_r16/xref/dalvik/libdex/DexFile.h)

* 简略结构如下：

```c++
/*
 * Direct-mapped "header_item" struct.
 */
struct DexHeader {
    u1  magic[8];           /* includes version number */
    u4  checksum;           /* adler32 checksum */
    u1  signature[kSHA1DigestLen]; /* SHA-1 hash */
    u4  fileSize;           /* length of entire file */
    u4  headerSize;         /* offset to start of next section */
    u4  endianTag;
    u4  linkSize;
    u4  linkOff;
    u4  mapOff;
    u4  stringIdsSize;
    u4  stringIdsOff;
    u4  typeIdsSize;
    u4  typeIdsOff;
    u4  protoIdsSize;
    u4  protoIdsOff;
    u4  fieldIdsSize;
    u4  fieldIdsOff;
    u4  methodIdsSize;
    u4  methodIdsOff;
    u4  classDefsSize;
    u4  classDefsOff;
    u4  dataSize;
    u4  dataOff;
};
```

* dex header重要属性

|      名称      |        格式        | 说明                                                         |
| :------------: | :----------------: | ------------------------------------------------------------ |
|     header     |    header_item     | 标头                                                         |
|   string_ids   |   string_id_item   | 字符串标识符列表。这些是此文件使用的所有字符串的标识符，用于内部命令（例如类型描述符）或用作代码引用的常量对象。此列表必须使用UTF-16代码点值按字符串内容进行排序（不采用语言区域敏感方式），且不得包含任何重复条目 |
|    type_ids    |    type_id_item    | 类型标识符列表。这些是此文件引用的所有类型（类、数组或原始类型）的标识符（无论文件中是否已定义）。此列表必须按"string_id"索引进行排序，且不包含任何重复条目 |
|   proto_ids    |   proto_id_item    | 方法原型标识符列表。这些是此文件引用的所有原型的标识符。此列表必须按返回类型（按"type_id"索引排序）主要顺序进行排序，然后按参数列表（按"type_id"索引排序的各个参数，采用字典排序方法）进行排序。且不包含任何重复条目 |
|   field_ids    |   field_id_item    | 字段标识符列表。这些是此文件引用的所有字段的标识符（文论文件中是否已定义）。此列表必须进行排序，其中定义类型（按"type_id"索引排序）是主要顺序，字段名称（按"string_id"索引排序）是中间顺序，而类型（按"type_id"索引排序）是次要顺序。该列表不的包含任何重复条目。 |
|   method_ids   |   method_id_item   | 方法标识符列表。这些是此文件引用的所有方法的标识符（无论文件中是否已经定义）。此列表必须进行排序，其中定义类型（按"type_id"索引排序）是主要顺序，方法名称（按"string_id"索引排序）是中间顺序，而方法原型（按索引排序"proto_id"）是次要顺序。该列表不得包含任何重复条目 |
|   class_defs   |   class_def_item   | 类定义列表。这些类必须进行排序，以便所指定类的超类和已实现的接口比引用类更早出现在该列表中。此外，对于在列表中多次出现的同类名，其定义是无效的。 |
| call_site_ids  | call_site_id_item  | 调用站点标识符列表。这些是此文件引用的所有调用站点的标识符（无论文件中是否已定义）。此列表必须按"call_site_off"的升序进行排序 |
| method_handles | method_handle_item | 方法句柄列表。此文件引用的所有方法句柄的列表（无论文件中是否已定义）。此列表未进行排序，而且可能包含将在逻辑上对应于不同方法句柄实例的重复项 |
|      data      |       ubyte        | 数据区，包含上面所列表格的所有支持数据。不同的项有不同的对齐要求；如果有必要，则在每个项之前插入填充字节，以实现所需的对齐效果 |
|   link_data    |       ubyte        | 静态链接文件中使用的数据。本文档尚未指定本区段中数据的格式。此区段在未链接文件中为空，而运行时实现可能会在适当的情况下使用这些数据。 |

