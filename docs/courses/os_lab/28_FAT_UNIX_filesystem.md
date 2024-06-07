# FAT 和 UNIX 文件系统

**本讲内容**：

- 文件系统实现分析
- FAT 文件系统
- UNIX 文件系统

## 磁盘上的数据结构

### 文件系统实现

文件的实现

- 文件 = “虚拟” 磁盘
- `API: read, write, ftruncate, ...`

目录的实现

- 目录 = 文件/目录的集合
- `API: mkdir, rmdir, readdir, link, unlink, ...`

思考题：`mount` 如何实现？

- 最好由操作系统统一管理 (而不是由具体的文件系统实现)

### 如果是《数据结构》课？

借助 RAM (Random Access Memory) 自由布局目录和文件

- `FileSystem` 就是一个 `Abstract DataType (ADT)`
- `Memory Hierarchy` 在苦苦支撑 “random access” 的假象

```c
class FSObject {
};

class File: FSObject {
  std::vector<char> content;
};

class Directory: FSObject {
  std::map<std::string,FSObject*> children;
};
```

- 实现上面的 API 完全就是个课后习题

### 回到《操作系统》课

没有 Random Access Memory

- 只有 <font color='red'>**block device**</font> 和两个 API（被迫读写连续的一块数据）
  - `bread(int bid, struct block *b);`
  - `bwrite(int bid, struct block *b);`

如果用 block device 模拟 RAM，就会出现**严重的读/写放大**

- 我们需要更适合磁盘的数据结构实现 `read`, `write`, `ftruncate`, ...

## File Allocation Table (FAT)

### 让时间回到 1980 年

5.25" 软盘：单面 180 KiB

- 360 个 512B 扇区 (sectors)
- 在这样的设备上实现文件系统，应该选用怎样的数据结构？

<img src=".pics/floppy-drives.jpg" alt="img" style="zoom:67%;" />

### 需求分析

相当小的文件系统

- 目录中一般只有**几个、十几个**文件
- 文件以**小文件**为主 (几个 block 以内)

文件的实现方式

- `struct block *`的链表
  - 任何复杂的高级数据结构都显得浪费

目录的实现方式

- 目录就是一个普通的文件 (虚拟磁盘；“目录文件”)
- 操作系统会对**文件的内容作为目录的解读**
  - 文件内容就是一个 `struct dentry[];`

### 用链表存储数据：两种设计

在**每个数据块后放置指针**

- **优点**：实现简单、无须单独开辟存储空间
- **缺点**：数据的大小不是 $2^k$; 单纯的 lseek 需要读整块数据

将**指针集中存放在文件系统的某个区域**

- **优点**：局部性好；lseek 更快
- **缺点**：集中存放的数据损坏将导致数据丢失

### 集中保存所有指针



## ext2/UNIX 文件系统

