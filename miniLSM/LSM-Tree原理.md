索引基于B树的数据库可以快速地单点查询、范围查询, 但写操作的log在落盘时会产生随机IO, 这导致如innoDB等引擎的写操作变慢.

参考kafka的AOF思想, 变随机IO为顺序IO, 以此为出发点改造CRUD操作和存储模型.

## 总体结构

`memtable`在内存中, 仍然基于b树存储少量数据
![[Pasted image 20230204222728.png]]

- 当`memtable`达到指定大小, 启动落盘进程: 遍历b树, 将结果存储为有序数组并清空`memtable`. 每次落盘形成1个`SSTable`
![[Pasted image 20230204223139.png]]

- 按照优化写操作的思路:
	- 读操作: 先查memtable, miss再对每一个SSTable进行二分搜索
	- 写操作: 使用追加写覆盖旧值, 删除单独定义`tombstone`值

- 但是随着数据量增加, 追加写也导致了SSTable中存在大量无法删除的旧值
![[Pasted image 20230204223818.png]]
* 注意在b树中也存在使用deleted标记被删除页, 导致的空洞问题. MySQL使用离线/在线重写技术使页结构紧凑化


