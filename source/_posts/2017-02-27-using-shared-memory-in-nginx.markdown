---
layout: post
title: "Nginx高级编程：在模块中使用共享内存"
date: 2017-02-27 10:02:23 +0800
comments: true
categories:  nginx
---

Nginx是由多个worker进程运行的，所以这些进程之间可以通过共享内存进行通信或完成一些进程间的工作，例如将用户session存储至nginx的内存中以提升运行效率等。Nginx提供了共享内存的API，由于其master-worker的模式，使得共享内存的使用和其与标准的内存分配略有不同。本文所使用的代码在这[nginx-shared-memory-module](https://github.com/friparia/nginx-shared-memory-module)，测试开发环境为nginx 1.10.1。本文参考了[Emiller's Advanced Topics In Nginx Module Development](http://www.evanmiller.org/nginx-modules-guide-advanced.html)。

<!--more-->

#创建并使用共享内存段(shared memory segment)

##注册共享内存的存储

和普通的模块开发类似，将所需的变量存储至配置的上下文中(可能会有更好的读取访问的位置)，结构体定义如下:

```c
typedef struct{
  ngx_shm_zone_t *shm_zone;
} ngx_http_hello_world_loc_conf_t;
```

其中使用了`ngx_shm_zone_t`结构体，即一个共享内存区，其声明如下：

```c
struct ngx_shm_zone_s {
  void *data;
  ngx_shm_t shm;
  ngx_shm_zone_init_pt init;
  void *tag;
};
```

`data`即共享内存中的自定义数据结构，初始化时使用。`shm`则为真正的共享内存。`init`为初始化函数的入口地址。`tag`则为一个标记指针。


##提供初始化函数

上节所提及的`init`通过如下函数进行实现：

```c
static ngx_int_t ngx_http_hello_world_init_shm_zone(ngx_shm_zone_t *shm_zone, void *data){
  /*  ...  */
  if(data){ 
    shm_zone->data = data;
    return NGX_OK;
  }
  /*  ...  */
  return NGX_OK;
}
```

##调用添加共享内存(`ngx_shared_memory_add`)函数

`ngx_shared_memory_add`函数为`shm_zone_t`分配内存，函数声明如下：

```c
ngx_shm_zone_t *
ngx_shared_memory_add(ngx_conf_t *cf, ngx_str_t *name, size_t size,
            void *tag);
```

一般在模块command入口或配置创建入口进行分配，该函数需要四个参数，`cf`为对应配置的上下文，`name`为共享内存的名称，如果已经存在相同名称的内存，则不会创建新的内存，直接返回此内存，`size`为此共享内存大小，一般来说为内存页大小的倍数即可，`tag`为标记，在模块中使用模块的指针做为`tag`。创建完该内存后，就应当配置共享内存的init入口和在配置中的初始化。

```c
shm_name = ngx_palloc(cf->pool, sizeof *shm_name);
shm_name->len = sizeof("shared_memory") - 1;
shm_name->data = (unsigned char *) "shared_memory";
shm_zone = ngx_shared_memory_add(cf, shm_name, 8 * ngx_pagesize, &ngx_http_hello_world_module);

if(shm_zone == NULL){
  return NGX_CONF_ERROR;
}

shm_zone->init = ngx_http_hello_world_init_shm_zone;
conf->shm_zone = shm_zone;
```

#使用slab分配内存

在`init`函数中，需要将实用的内存进行初始化，则需要`ngx_slab_alloc`函数进行分配，完善后的`init`函数如下：

```c
static ngx_int_t ngx_http_hello_world_init_shm_zone(ngx_shm_zone_t *shm_zone, void *data){
  ngx_slab_pool_t *shpool;
  ngx_http_hello_world_shm_count_t *shm_count;
  if(data){ 
    shm_zone->data = data;
    return NGX_OK;
  }
  shpool = (ngx_slab_pool_t *)shm_zone->shm.addr;
  shm_count = ngx_slab_alloc(shpool, sizeof *shm_count);
  shm_count->count = 0;
  shm_zone->data = shm_count;
  return NGX_OK;
}
```

从`shm_zone`中获取共享内存的地址做为内存池，然后调用`ngx_slab_alloc`函数从此内存池中分配一块所需的内存，再对这块内存进行初始化。

除了`ngx_slab_alloc`来分配内存，我们还需要`ngx_slab_free`函数进行内存的释放。

#内存的原子访问

在多进程中同时访问一块内存是非常危险的一件事情。Nginx的API也提供了原子锁机制(`shpool->mutex`)来保证内存的一致性。Nginx提供了`ngx_shmtx_lock(&shpool->mutex)`和`ngx_shmtx_ulock(&shpool->mutex)`对锁进行操作。如果进行分配内存、销毁内存时，就需要用到锁进行操作。因为`mutex`互斥锁使用spinlocks实现，很容易用掉100%的CPU，所以不要在获得`mutex`互斥锁的时候，执行一些耗时较长的操作。

```c
shpool = (ngx_slab_pool_t *) lccf->shm_zone->shm.addr;
ngx_shmtx_lock(&shpool->mutex);
new_block = ngx_slab_alloc_locked(shpool, ngx_pagesize);
ngx_shmtx_unlock(&shpool->mutex);
```

