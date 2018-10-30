# 解析Instagram网站的图片存储架构

这篇文章主要介绍了Instagram网站的图片存储架构,主要由Python的Django驱动的Instagram后台在PostgreSQL和Redis数据存储的使用方面同样亮点颇多,需要的朋友可以参考下

被Facebook以10亿美金收购的著名手机照片分享应用Instagram最近吸引了无数人的眼球，Instagram联合创始人Mike Krieger说他们用了8周时间打造了最初的Instagram，但现在的系统肯定已经今非昔比。Instagram技术团队曾发表过一篇文章，介绍了Instagram背后的技术，日前Mike Krieger在名为Scaling Instagram的演讲里，又介绍了更多细节，让人们能了解到5名技术人员是如何支撑起整个系统的。

一张照片上传的过程是这样的：

**1.采用同步的方式写入媒体数据库**

**2.如果照片上有地理位置标签，则以异步的方式将照片提交给Solr进行索引**

**3.将照片的ID加入每个关注者的列表里，该列表保存在Redis之中**

**4.在显示Feed时，选取一小部分照片ID，在Memcached里进行查询**

**5.在设计系统时，Instagram的设计哲学是简单、为最小化运维负担进行优化并监控一切内容；其核心原则是保持简单，不要重复发明轮子，尽可能使用经过验证、稳定可靠的技术。**

由于只有5名技术人员（其中仅2.5名后端工程师），精力有限，选择Amazon的云服务是个不错的选择。目前他们使用了超过100个EC2实例用于提供各种服务，运行的操作系统是Ubuntu 11.04，之前的一些版本在高流量时表现不够稳定。在负载均衡方面，他们使用Amazon的Elastic Load Balancer实现负载均衡，后端运行了3个Nginx实例，SSL只到ELB上为止，降低了Nginx上的CPU负载。DNS和CDN分别由Amazon的Route 53和CloudFront提供，所有的照片都存放在S3上，目前已经有几TB的规模了。

用于处理请求的应用服务器运行于Amazon High-CPU Extra-Large Instance之上，由于他们的请求更多是CPU密集型的，因此这能更好地平衡CPU与内存。采用的开发框架是Django，WSGI服务器是Gunicorn，通过Fabric在所有机器上进行并行部署，一次部署仅需几秒钟。

用户信息、图片元数据、标签等大部分数据存储在 PostgreSQL 中。 

实践中发现 Amazon 的网络磁盘系统单位时间内寻道能力不行，所以有必要将数据尽量放到内存中。创建了软 RAID 以提升 IO 能力，使用的 Mdadm 工具进行 RAID 管理。

管理内存中的数据，vmtouch 这个小工具值得推荐。

PostgreSQL 设置为 Master-Replica 方式，流复制模式。利用 EBS 的快照进行数据库备份。使用 XFS 文件系统，以便和快照服务充分配合。 使用 repmgr 这个小工具做 PostgreSQL 复制管理器器。

连接池管理，用了 Pgbouncer。Christophe Pettus 的文章包含了不少 PostgreSQL 数据库的信息。

应用程序在连接数据库时，由Pgbouncer建立连接池。目前，Instagram的数据按照用户ID进行分片，某些分片可能会超出物理节点的容量上限，为此他们将数据分成了很多个逻辑分片，映射到少数几个物理节点之上；当一个节点被填满之后，可以将某些逻辑分片移到别的节点上，以缓解该节点的压力。随着数据量的增长，以后他们也会进行垂直分区，Django DB Router能让一切轻松不少。

Instagram也大量使用Redis来存放复杂的对象（对象的大小做了一定的限制），用于主Feed、活动Feed、会话系统及其他相关系统。因为要将Redis的所有数据都放在内存里，此处同样也用了High-Memory Quadruple Extra-Large Instance，并对数据做了分片。当Redis实例的请求达到4万/秒后，它渐渐成为了瓶颈，于是Redis也做了主从复制，副本的数据会经常导出到磁盘上，通过EBS快照进行备份。

除了Redis，他们还使用Memcached来做缓存，目前运行了6个实例，应用服务器通过pylibmc和libmemcached进行连接。虽然Amazon提供了Elastic Cache服务，但该服务的价格并不便宜，相比之下，还是运行自己的Memcached实例比较划算。异步任务队列使用的是Gearman，目前有大约200个工作进程来处理各种任务，比如把照片分享到Twitter和Facebook，通知用户有新照片等等。Pyapns已经处理了十亿的推送通知，非常稳定，他们还自己开发了基于Node.js的node2dm，用于向Android设备发送推送通知。

监控方面，Instagram使用Munin以图形化的方式呈现整个系统的运行状况，还通过Python-Munin定制了一些插件，用来显示业务数据；网络守护进程Stated可以实时收集数据并做汇总；Dogslow会监控进程，一旦发现运行时间过长的进程，便会保存该进程的快照，以便后续分析，比如响应时间超过1.5秒的请求，通常都是卡在Memcached的set()和get_many()方法上。对于Python的错误，只要登上Sentry就能实时获取错误信息。

HighScalability上还根据整理Instagram团队软件工程师Mike Krieger的演讲整理了一些值得借鉴的经验，比如：

**1.找那些你熟悉的技术和工具，在简单的使用场景里先做一些尝试**

**2.不要使用两个工具来处理同样的任务**

**3.事先准备降级方案，以便在需要时降低负载**

**4.不要过度优化，或者希望能事先知道站点要扩展，对于一个初创的社交站点而言，没什么扩展性问题是解决不了的**

**5.如果一个办法不行，赶快换下一个**

![](https://i01picsos.sogoucdn.com/c5966240167a5580)
