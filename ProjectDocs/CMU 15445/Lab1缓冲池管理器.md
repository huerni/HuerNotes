lab1的目的是实现一个缓冲池管理器。该缓冲池又由可扩展哈希表和lru-k替换策略实现。  


## 可扩展哈希表

可扩展哈希表相比哈希表，支持自动扩展大小。在使用可扩展哈希表时，用户无需关心哈希表的负载因子是否过大，哈希表容量是否需要扩充，这一切都由可扩展哈希表自己处理。  

可扩展哈希表由许多桶组成，即采用链表法解决冲突，冲突的key都存放在一个桶内。当桶内的数据数量过多时，桶就会进行分裂，以分别装进这些数据。  

我们要实现的功能主要有：
1. 通过key查到对应的桶，然后对该桶进行插入，删除和查询    
2. 插入桶时，我们需要实现桶分裂的一些细节，包括桶分裂kv数据重新分配，目录dir扩展，桶ID重新分配。   

第一点比较容易，代码已经实现了`IndexOf`方法， 用于实现输入key哈希到对应的index，实际上就是求出key二进制的前`global_depth`位作为index。然后在桶内list遍历删除对应key即可。  

第二点，当桶内大小超过设置的最大`size_`时，需要进行分裂。 桶分裂时，增加桶的`local_depth`，然后重新分配桶内kv对。  

**分裂后，怎么将key准确地重分配进桶内？**  

一个桶内，所有的key前`local_depth`位是相等的，当`local_depth+1`时，第`local_depth+1`位可能不一样，所以kv数据有可能分成两份，分别存入分裂的两个桶内。**这里我的方法并不是最佳方法，甚至可能是费时的方法，但能通过测试**。首先保存桶内所有数据，然后遍历dir，由于`local_depth+1`，指向该桶的所有目录指针也会分成两份，所以创建两个新桶出来，重新分配指向待分裂的桶的指针。最后将list中所有key重新插入一遍。   

**目录dir扩展，桶ID重新分配是怎么回事？**
`local_depth`总是小于`globa_depth`的，当桶的`local_depth`与`global_depth`相等时，此时桶再分裂可能比目录的长度还多了，所以目录需要扩充，这里是扩充两倍，同时将`global_depth+1`，然后针对新的`global_depth`，重新分配指针指向桶。  


## LRU-K替换策略

LRU-K替换策略是LRU的升级版，相比于LRU替换算法，它解决了缓存污染问题。  

LRU-K对每一帧数据都记录了最后k个访问时间，替换时首先替换没有访问K次的帧，其次是第k次访问最早的帧。  

这部分主要实现4个功能，Evict，RecordAccess，SetEvictable，Remove。


### Evict

该方法选择需要替换的帧进行替换，根据以下规则：
1. 替换的帧必须是可替换的，即是is_evictable_的。
2. 按照进入时间先后顺序遍历，当帧没有k次访问历史时，即为初始值INF，进行替换  
3. 否则，找到第k次访问最早的帧，进行替换  

### RecordAccess  

1. 判断帧是否存在，如果存在，更新访问时间，返回结果。    
2. 若不存在，且插入后大小超过最大size，按照Evict同样的逻辑进行替换。  
3. 插入，返回结果。  

### SetEvictable

将不可替换的帧设置为可替换，注意，当前大小`curr_size_`指的是可替换帧的数目。  

### Remove

将可替换帧删除即可。lab描述中有提到，如果删除不可替换帧，需`abort()`终止程序。  


## BufferPool Manager  

最后，我们要基于上述两个组件实现缓冲池管理器。  


缓冲池管理器有`page_table_`，保存读取的页面id和帧id之间的映射；`free_list_`存储所有空闲的帧。初始化时，缓冲池为空，即`page_table_`为空，`free_list_`存有所有空闲帧，帧id分别为`0~pool_size_`。  

我们要实现NewPgImp、FetchPgImp、UnpinPgImp、FlushPgImp、FlushAllPgsImp、DeletePgImp几个功能。  

### NewPaImp

该方法实现创建一个新的`page`页面，并存入缓冲池中，有以下几个步骤：

1. 首先判断是否有空帧，即`free_list_`是否为空，若不为空，则从`free_list_`中取出空帧存放该`page`。  
2. 否则，就该选择替换帧来存放该`page`。选择替换帧直接使用上文实现的LUR-K替换策略找到替换帧，**如果该帧中存放的`page`是脏页，则需要写入到持久化磁盘中**。  
3. 找到可用帧后，初始化`page`页面(查看`Page`类)，然后放入该帧中。  

管理器用`pin_count`表示`page`页面是否被引用，当该`page`页面被引用是，表示正在被使用，不能被替换或删除，所以该`page`对应的帧`Evictable`应该设置为false。当`pin_count`为0时，才能设置为true，这正是`UnpinPgImp`实现的功能。

### FetchPgImp

FetchPage()表示取出一个页面进行使用。它首先在缓冲区进行查找，如果找不到，从磁盘读取，具体步骤如下：  

1. 从page_table中找到，找到后直接返回该page供使用。注意此时`pin_count+1`。    
2. 不在缓冲区中，找到可替换帧，逻辑与NewPage相同，然后从磁盘中读取Page(调用ReadPage)，存入缓冲区中返回。  

### FlushPgImp和FlushAllPgsImp

FlushPgImp和FlushAllPgsImp实现比较简单，flushPage将指定page写入到磁盘，而flushAllPage将缓冲区中所有page写入到磁盘。注意写入后`is_dirty_`为false。  


### DeletePgImp

找到指定的page，将其从缓冲区中删除。注意，`pin_count>0`不可删除  

## 参考资料
https://segmentfault.com/a/1190000022558044  
https://15445.courses.cs.cmu.edu/fall2022/project1/






