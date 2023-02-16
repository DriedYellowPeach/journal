| Author | 王润泽                 |
| ------ | ---------------------- |
| Date   | 2023-2-13              |
| Email  | wangrunze13@huawei.com |

# 1. 方案目标
目标是把isulad当前的layer store里的RO层分离出来，通过远程文件共享(nfs)让多台启动的isulad实例共享某些只读的数据；具体来说，在两个host A和B上都启动了iSulad, 如果A pull或者load了镜像busybox, 那么B上的isulad同样可以使用这个镜像。

```
operations:

+--------------------+           +--------------------+           +--------------------+
| isula pull busybox |           | without pull       |           | without pull       |
| isula pull nginx   |           | isula run busybox  |           | isula run ngxinx   |
| isula pull ...     |           | isula run ...      |           | isula run centos   |
+--------------------+           +--------------------+           +--------------------+
    |                                |                                |
    ▼                                ▼                                ▼
+======================+         +======================+         +======================+
| isulad on Host A     |         | isulad on Host B     |         | isulad on Host C     |
+======================+         +======================+         +======================+
| image store module   |         | image store module   |         | image store module   |
+----------------------+         +----------------------+         +----------------------+
| refresh thread off   |         | refresh thread on    |         | refresh thread on    |
+----------------------+         +----------------------+         +----------------------+
| local rw | remote ro |         | local rw | remote ro |         | local rw | remote ro |
+----------------------+         +----------------------+         +----------------------+
                |                               |                             |            
                | enable nfs                    | mounted on                  | mounted on
                ▼                               ▼                             ▼           
            +=====================================================================+
            |                    nfs directory over network                       |
            +=====================================================================+
            |                          image store                                |
            +---------------------------------------------------------------------+
            |   image      |   image     |   image   |   image     |   image      | 
            |   busybox    |   nginx     |   my-app  |   ubuntu    |   centos     | 
            |   4MB        |   100MB     |   1.2GB   |   5MB       |   6MB        | 
            +---------------------------------------------------------------------+
            |                          layers store                               |
            +---------------------------------------------------------------------+
            |     layer            |      layer            |        layer         |
            |   05c361054          |    8a9d75caad         |      789d605ac       |
            +---------------------------------------------------------------------+


```

# 2. 总体设计
总体来说有两部分的功能：
- iSulad原有的image storage适配分离的RO目录结构，*分离的RO目录*可用于远程挂载
- iSulad实例同步内存数据，镜像数据和layer数据*定期更新*，不重启isulad即可使用由其他isulad实例新增的镜像 

*分离RO目录*
修改前后storage目录结构对比：

```
old:
overlay-layer
├── b703587
│    └── layer.json
└── b64792c
     └── layer.json 
     └── b64792.tar.gz

new:
overlay-layer
├── b64792c -> ../RO/b64792c
├── b703587
│    └── layer.json
└── RO
    └── b64792c
	      └── layer.json
          └── b64792.tar.gz
```

以overlay-layers目录为例，创建新layer时，如果是只读层，就把层数据放到RO目录下，在RO上层目录创建软连接指向真实数据。删除layer时需要额外删除软连接。


*定期更新*
定期更新通过启动一个线程按照固定的周期扫描`overlay-image`目录，如果发现新增或者删除的镜像，就触发更新image和layer内存数据的流程，更新过后，各个iSulad实例之间可以使用相同的镜像。

```
+---------------------+          +---------------------+           +-----------------------+
| image remote module |          | layer remote module |           | overlay remote module |
+---------------------+          +---------------------+           +-----------------------+
         |                                  |                                  |
         | refresh start                    |                                  |
         | image_dir_scan                   |                                  |
         |                                  |                                  |
         |       pending_load_layers        |                                  |
         |---------------------------------▶|                                  |
         |                                  |                                  |
         |                                  |--▶pending top layer id           |
         |                                  |             |                    |
         |                                  |             |  converting        |
         |                                  |             |                    |
         |                                  |             ▼                    |
         |                                  |◀--full list of pending layer     |
         |                                  |                                  |
         |                                  |         all pending layer        |
         |                                  |---------------------------------▶|
         |                                  |                                  |   maintain symbol link
         |                                  |                                  |   to real overlay ro layer
         |                                  |                                  |   overlay loading done
         |                                  |       overlay load success       |
         |                                  |◀---------------------------------|
         |                                  |                                  |
         |                                  |  insert layer into map           |
         |                                  |  maintain symbol link            |
         |                                  |  to real ro layer                |
         |                                  |  layer loading done              |
         |                                  |                                  |
         |       layer load success         |                                  |
         |◀---------------------------------|                                  |
         |                                  |                                  |
         | insert new image into map        |                                  |
         | image loading done               |                                  |
         | image load success               |                                  |
         | refresh success                  |                                  |
         |                                  |                                  |
+---------------------+          +---------------------+           +-----------------------+
| image remote module |          | layer remote module |           | overlay remote module |
+---------------------+          +---------------------+           +-----------------------+

```

*同步问题*
共享资源并发使用发生竞争的条件是：`two process access the same resource concurrently and at least one of the access is a writer`。这里的共享资源有：

```
+============================+         +=====================================+
|      Sharing Resource      |         |          Storage                    |
+============================+         +=====================================+
| read-only overlay layers   | mounted | /var/lib/isulad/overlay/RO          |
+----------------------------+ ======▶ +-------------------------------------+
| reald-only layers metadata | shared  | /var/lib/isulad/overlay-layers/RO   |
+----------------------------+         +-------------------------------------+
| reald-only images          |         | /var/lib/isulad/overlay-images      |
+----------------------------+         +-------------------------------------+

```
而分布在不同host上的isulad进程通过网络共享这些资源，如果不考虑新增删除的情况，不会出现资源竞争，所有的节点都是reader。

对于主节点pull镜像，其他节点使用镜像的情况，主节点是writer，其他节点是reader。这时候可能出现的问题是主节点pull镜像的流程没有完全结束，但是其他节点开始使用这个不完整的镜像。对于这个问题的解决方案是通过扫描image目录来新增镜像，通过这种方式能确保一定该新增镜像的信息完整。

```
+---------------------+         +--------------------+         +-----------------------+         +-----------------------+
|    registry         |         | image module       |         | layer store module    |         | driver overlay module |
+---------------------+         +--------------------+         +-----------------------+         +-----------------------+
        |                                |                                 |                                 |
        |                                |                                 |                                 |
        | registry_pull                  |                                 |                                 |
        | fetch_manifest                 |                                 |                                 |
        | check reuse and fetch          |                                 |                                 |
        | +----------------+             |                                 |                                 |
        | | register_layer |             |                                 |                                 |
        | +----------------+             |                                 |                                 |
        |-----------------------------------------------------------------▶|                                 |
        |                                |                                 | layer_store_create              |
        |                                |                                 |--------------------------------▶|
        |                                |                                 |                                 | driver_create_layer
        |                                |                                 |                                 | +--------------------+ 
        |                                |                                 |                                 | | setup overlay dir  |
        |                                |                                 |                                 | +--------------------+
        |                                |                                 |                                 | driver_create_layer done
        |                                |                                 |◀--------------------------------|
        |                                |                                 | +------------+                  |
        |                                |                                 | | save layer |                  |
        |                                |                                 | +------------+                  |
        |                                |                                 | layer create done               |
        |◀-----------------------------------------------------------------|                                 |
        | all layer setup                |                                 |                                 |
        | +----------------+             |                                 |                                 |
        | | register image |             |                                 |                                 |
        | +----------------+             |                                 |                                 |
        |-------------------------------▶|                                 |                                 |
        |                                | storage_img_create              |                                 |
        |                                | set img top layer               |                                 |
        |                                | img create                      |                                 |
        |                                | +------------+                  |                                 |
        |                                | | save image |                  |                                 |
        |                                | +------------+                  |                                 |
        |                                | create image done               |                                 |
        |◀-------------------------------|                                 |                                 |
        | pull img done                  |                                 |                                 |
        |                                |                                 |                                 |
        |                                |                                 |                                 |
+---------------------+         +--------------------+         +-----------------------+         +-----------------------+
|    registry         |         | image module       |         | layer store module    |         | driver overlay module |
+---------------------+         +--------------------+         +-----------------------+         +-----------------------+

```

至于主节点删除镜像的情况，主节点是writer，其他节点是reader，可能出现的情况是其他节点还有容器在使用镜像的时候，镜像被删除，但是根据需求场景暂不处理这种情况。其他的处理与新增镜像相同，依然以image dir作为入口，扫描发现删除的镜像。删除时需要关注layer和overlay目录下的软链接。

# 3. 接口描述

```c
// 初始化remote模块里的layer data
int remote_layer_init(const char *root_dir);

// 初始化remote模块里的overlay data
int remote_overlay_init(const char *driver_home);

// 清理remote模块的资源
void remote_maintain_cleanup();

// 启动 定期更新的thread
int start_refresh_thread(void);

// 创建新layer目录
int remote_layer_build_ro_dir(const char *id);

// 创建新overlay目录
int remote_overlay_build_ro_dir(const char *id);

// 删除layer目录
int remote_layer_remove_ro_dir(const char *id);

// 删除overlay目录
int remote_overlay_remove_ro_dir(const char *id);
```

# 4. 详细设计
分离RO目录的关键在于适配原来的代码逻辑，原先的代码在操作镜像和层的时候，不管是RO层还是RW层，从创建到删除都是在当前目录下进行的，这就是我们额外创建一个软连接的作用:
- RO目录的作用是为了支持远程挂载
- 软连接的作用是模拟原来的目录结构

这样以来，image module的逻辑几乎不需要改动，除了以下几点需要注意：
- 创建和删除的时候需要处理一个额外的资源：软连接，之前只需要关注目录即可，现在如果创建的是只读层，就需要额外创建软连接，如果删除的是只读层，就需要额外删除软连接
- 以`overlay-layers`目录为例，isulad启动时会以正则规则扫描当前目录下的子目录是否合法，所以需要屏蔽`RO`目录


定时刷新的入口是`image_store`, 如果没有定时刷新，则isulad内存数据不会更新，这时即使磁盘上有完整的image和layer数据也不能使用，因为`isula images`是看不到新镜像的，而定时刷新也需要在layer和overlay层面实现，layer涉及到内存数据的更新以及维护额外的软连接，而对于overlay来说只需要维护额外的软连接，只有image，layer，overlay三个模块的刷新动作都完成了之后，才能利用新的镜像来创建启动容器

这里是各个模块定时刷新的具体逻辑：
- image模块：
    - scan image directory: image 目录是远程挂载的，通过扫描目录与内存中的image对比，可以发现哪些image是新增的，哪些是删除的, 将修改的这些信息整理成top layer list 传递给layer模块。
    - update image map: 等下层的layer模块和overlay模块处理完毕，则可以确定image成功增加或者删除，这时候更新image store map, 之后用`isula images`命令看到的就是远程的最新状态
- layer模块
    - handle top layers: 需要利用top layer id + layer序列化后的parent id找到所有的需要新增或者删除的layer，将这些layer传递给overlay模块
    - 根据需要新增或者删除的layer维护软链接
- overlay模块
    - 根据全量的layer信息维护软链接
