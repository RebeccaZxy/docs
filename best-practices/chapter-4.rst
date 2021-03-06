.. include:: ../defines.hrst

内存和负载管理
=================

内存管理
-----------

内存管理对 |product-name| 集群性能有显著影响。默认设置可以满足大多数环境需求。不要修改默认设置，除非你理解系统的内存特性和使用情况。如果精心设计内存管理，大多数内存溢出问题可以避免。

下面是 |product-name| 内存溢出的常见原因：

* 集群的系统内存不足
* 内存参数设置不当
* 段数据库(Segment)级别的数据倾斜
* 查询级别的计算倾斜

有时不仅可以通过增加系统内存解决问题，还可以通过正确的配置内存和设置恰当的资源队列管理负载，以避免很多内存溢出问题。

建议使用如下参数来配置操作系统和数据库的内存：

* vm.overcommit_memory

    这是 /etc/sysctl.conf 中设置的一个Linux内核参数。总是设置其值为2。它控制操作系统使用什么方法确定分配给进程的内存总数。对于 |product-name|，唯一建议值是2。

* vm.overcommit_ratio

    这是 /etc/sysctl.conf 中设置的一个Linux内核参数。它控制分配给应用程序进程的内存百分比。建议使用缺省值50.

* 不要启用操作系统的大内存页

* gp_vmem_protect_limit

    使用 *gp_vmem_protect_limit* 设置段数据库(Segment)能为所有任务分配的最大内存。切勿设置此值超过系统物理内存。如果 *gp_vmem_protect_limit* 太大，可能造成系统内存不足，引起正常操作失败，进而造成段数据库故障。如果*gp_vmem_protect_limit* 设置为较低的安全值，可以防止系统内存真正耗尽；打到内存上限的查询可能失败，但是避免了系统中断和Segment故障，这是所期望的行为。*gp_vmem_protect_limit* 的计算公式为:

::

        (SWAP + (RAM * vm.overcommit_ratio)) * .9 / number_segments_per_server

* runaway_detector_activation_percent

    |product-name| 提供了失控查询终止（Runaway Query Termination）机制避免内存溢出。系统参数 *runaway_detector_activation_percent* 控制内存使用达到 *gp_vmem_protect_limit* 的多少百分比时会终止查询，默认值是 90%。如果某个Segment使用的内存超过了 *gp_vmem_protect_limit* 的90%（或者其他设置的值），|product-name| 会根据内存使用情况终止那些消耗内存最多的 SQL 查询，直到低于期望的阈值。

* statement_mem

    使用 *statement_mem* 控制 Segment 数据库分配给单个查询的内存。如果需要更多内存完成操作，则会溢出到磁盘（溢出文件，spill files）。*statement_mem* 的计算公式为:

::

        (vmprotect * .9) / max_expected_concurrent_queries

    *statement_mem* 的默认值是128MB。例如使用这个默认值， 对于每个服务器安装8个Segment的集群来说，一条查询在每个服务器上需要1GB内存（8 Segments * 128MB）。对于需要更多内存才能执行的查询，可以设置回话级别的*statement_mem*。对于并发度比较低的集群，这个设置可以较好的管理查询内存使用量。并发度高的集群也可以使用资源队列对系统运行什么任务和怎么运行提供额外的控制。

* gp_workfile_limit_files_per_query

    *gp_workfile_limit_files_per_query* 限制一个查询可用的临时溢出文件数。当查询需要比分配给它的内存更多的内存时将创建溢出文件。当溢出文件超出限额时查询被终止。默认值是0，表示溢出文件数目没有限制，可能会用光文件系统空间。

* gp_workfile_compress_algorithm

    如果有大量溢出文件，则设置 *gp_workfile_compress_algorithm* 对溢出文件压缩。压缩溢出文件也有助于避免磁盘子系统I/O操作超载。

配置资源队列
---------------

|product-name| 的资源队列提供了强大的机制来管理集群的负载。队列可以限制同时运行的查询的数量和内存使用量。当 |product-name| 收到查询时，将其加入到对应的资源队列，队列确定是否接受该查询以及何时执行它。

* 不要使用默认的资源队列，为所有用户都分配资源队列。 每个登录用户（角色）都关联到一个资源队列；用户提交的所有查询都由相关的资源队列处理。如果没有明确关联到某个队列，则使用默认的队列 *pg_default*。

* 避免使用 gpadmin 角色或其他超级用户角色运行查询 超级用户不受资源队列的限制，因为超级用户提交的查询始终运行，完全无视相关联的资源队列的限制。

* 使用资源队列参数 *ACTIVE_STATEMENTS* 限制某个队列的成员可以同时运行的查询的数量。

* 使用 *MEMORY_LIMIT* 参数控制队列中当前运行查询的可用内存总量。联合使用 *ACTIVE_STATEMENTS* 和*MEMORY_LIMIT* 属性可以完全控制资源队列的活动。

队列工作机制如下：假设队列名字为 *sample_queue*，*ACTIVE_STATEMENTS* 为10，*MEMORY_LIMIT* 为 2000MB。这限制每个段数据库(Segment)的内存使用量约为2GB。如果一个服务器配置8个Segments，那么一个服务器上，*sample_queue* 需要16GB内存。如果Segment服务器有64GB内存，则该系统不能超过4个这种类型的队列，否则会出现内存溢出。（4队列 * 16GB/队列）。

注意 *STATEMENT_MEM* 参数可使得某个查询比队列里其他查询分配更多的内存，然而这也会降低队列中其他查询的可用内存。

* 资源队列优先级可用于控制工作负载以获得期望的效果。具有 *MAX* 优先级的队列会阻止其他较低优先级队列的运行，直到 *MAX* 队列处理完所有查询。

* 根据负载和一天中的时间段动态调整资源队列的优先级以满足业务需求。根据时间段和系统的使用情况，典型的环境会有动态调整队列优先级的操作流。可以通过脚本实现工作流并加入到 crontab 中。

* 使用 *gp_toolkit* 提供的视图查看资源队列使用情况，并了解队列如何工作。
