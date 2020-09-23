---
title: "Envoy源码分析之ThreadLocal机制"
linkTitle: "Envoy源码分析之ThreadLocal机制"
date: 2020-08-23
description: Envoy中的`ThreadLocal`机制其实就是我们经常说的线程本地存储简称TLS(Thread Local Storage)，顾名思义通过TLS定义的变量会在每一个线程专有的存储区域存储一份，访问TLS的时候，其实访问的是当前线程占有存储区域中的副本，因此可以使得线程可以无锁的并发访问同一个变量。
---

# ThreadLocal机制

​	Envoy中的`ThreadLocal`机制其实就是我们经常说的线程本地存储简称TLS(Thread Local Storage)，顾名思义通过TLS定义的变量会在每一个线程专有的存储区域存储一份，访问TLS的时候，其实访问的是当前线程占有存储区域中的副本，因此可以使得线程可以无锁的并发访问同一个变量。Linux上一般有三种方式来定义一个TLS变量。

* gcc对C语言的扩展`__thread`
* pthread库提供的`pthread_key_create`
* C++11的`std::thread_local`关键字

​	Envoy的`ThreadLocal`机制就是在C++11的`std::thread_local`基础上进行了封装用于实现线程间的数据共享。Envoy因其配置的动态生效而出名，而配置动态生效的基石就是`ThreadLocal`机制，通过`ThreadLocal`机制将配置可以无锁的在多个线程之间共享，当配置发生变更的时候，通过主线程将更新后的配置Post到各个线程中，交由各个线程来更新自己的`ThreadLocal`。


# ThreadLocalObject

​	Envoy要求所有的`ThreadLocal`数据对象都要继承`ThreadLocalObject`，比如下面这个`ThreadLocal`对象。

```cpp
struct ThreadLocalCachedDate : public ThreadLocal::ThreadLocalObject {
   ThreadLocalCachedDate(const std::string& date_string) : 
   date_string_(date_string) {}
  const std::string date_string_;
};
```

​	但实际上`ThreadLocalObject`只是一个空的接口类，所以并非我们继承了`ThreadLocalObject`就是一个TLS了。继承`ThreadLocalObject`目的是为了可以统一对所有要进行TLS的对象进行管理。

```cpp
class ThreadLocalObject {
public:
  virtual ~ThreadLocalObject() = default;
};
using ThreadLocalObjectSharedPtr = std::shared_ptr<ThreadLocalObject>;
```

​	Envoy中需要TLS的数据有很多，最重要的当属配置，随着配置的增多，这类数据所占据的内存也会变得很大，如果每一种配置都声明为TLS会导致不少内存浪费。为此Envoy通过`ThreadLocalData`将所有要进行TLS的对象都管理起来，然后将`ThreadLocalData`本身设置为TLS，通过TLS中保存的指针来访问对应的数据。这样就可以避免直接在TLS中保存数据而带来内存上的浪费，只需要保存指向数据的指针即可，相关代码如下。

```cpp
struct ThreadLocalData {
  // 指向当前线程的Dispatcher对象 
  Event::Dispatcher* dispatcher_{};
  // 保存了所有要TLS的数据对象的智能指针，通过智能指针来访问真正的数据对象
  std::vector<ThreadLocalObjectSharedPtr> data_;
};
```

![4-2.jpg](https://ata2-img.oss-cn-zhangjiakou.aliyuncs.com/a4712346b33dc89bc09bb5c41332c5ba.jpg)

 	如上图所示，每一个TLS通过指针指向实际的对象，每一个数据对象只在内存中保存一份，避免内存上的浪费，但是这样带来问题就是如何做到线程安全的访问数据对象呢?  当我们要访问数据对象的时候，如果此时正在对数据对象进行更新，这个时候就会存在一个线程安全的问题了。Envoy巧妙的通过在数据对象更新的时候，先构造出一个新的数据对象，然后将TLS中的数据对象指针指向新的数据对象来实现线程安全的访问。本质上和COW(copy-on-write)很类似，但是存在两点区别。

* COW中是先拷贝原来的对象，然后更改对象，而Envoy在这里是重新构建一个新的数据对象
* COW中无论是读还是写，在更改`shared_ptr`指向时，都需要加锁，因为`shared_ptr`本身的读写时非线程安全的，而Envoy不需要加锁。



​	Envoy中指向数据对象的`shared_ptr`并非只有一个，而是每一个线程都有一个`shared_ptr`指向数据对象，更改`shared_ptr`指向新的数据对象时通过post一个任务到对应线程中，然后在同一个线程使`shared_ptr`指向新的数据对象，因此并没有多线程操作`shared_ptr`，所以没有线程安全问题，自然也不用加锁，这是Envoy实现比较巧妙的地方。

![4-3.jpg](https://ata2-img.oss-cn-zhangjiakou.aliyuncs.com/b1b456618697f8df6c2a0e81bba4a907.jpg)


​	如上图所示，T1时刻，Thread1通过TLS对象访问`ThreadLocalObjectOld`，在T2时刻在main线程发现配置发生了变化，重新构造了一个新的`ThreadlocalObjectNew`对象，然后通过Thread1的`Dispatcher`对象post了一个任务到Thread1线程，到了T3时刻这个任务开始执行，将对应的指针指向了 `ThreadLocalObjectNew`，最后在T4时刻再次访问配置的时候，就已经访问的是最新的配置了。到此为止就完成了一次配置更新，而且整个过程是线程安全的。


# ThreadLocal

​	终于到了分析真正的ThreadLocal对象的时候，它的功能其实很简单，大部分的能力都是依赖`Dispatcher`、还有上文中提到的`SlotImpl`、`ThreadLocalData`等，`Instance`是它的接口类，它继承了`SlotAllocator`接口，也包含了上文中分析的`allocateSlot`方法。

```cpp
class Instance : public SlotAllocator {
public:
  // 每启动一个worker线程就需要通过这个方法进行注册
  virtual void registerThread(Event::Dispatcher& dispatcher, bool main_thread) PURE;
  // 主线程在退出的时候调用，用于标记shutdown状态
  virtual void shutdownGlobalThreading() PURE;
  // 每一个worker线程需要调用这个方法来释放自己的TLS
  virtual void shutdownThread() PURE;
  virtual Event::Dispatcher& dispatcher() PURE;
};
```

​	对应的实现是`InstanceImpl`对象，在`Instance` 的基础上又扩展了一些post任务到所有线程的一些方法。

```cpp

class InstanceImpl : public Instance {
 public:
	....
 private:
  // post任务到所有注册的线程中
  void runOnAllThreads(Event::PostCb cb);
  // post任务到所有注册的线程中，完成后通过main_callback进行通知
  void runOnAllThreads(Event::PostCb cb, Event::PostCb main_callback);
  // 初始化TLS指向对应的数据对象指针
  static void setThreadLocal(uint32_t index, ThreadLocalObjectSharedPtr object);
  .....
  // 保存所有注册的线程
  std::list<std::reference_wrapper<Event::Dispatcher>> registered_threads_;
```

​	因为所有的线程都会注册都`InstanceImpl`中，所以只需要遍历所有的线程所对应的`Dispatcher` 对象，调用其post方法将任务投递到对应线程即可，但是如何做到等所有任务执行完成后进行通知呢 ？

```cpp
void InstanceImpl::runOnAllThreads(Event::PostCb cb, 
                                   Event::PostCb all_threads_complete_cb) {
  ASSERT(std::this_thread::get_id() == main_thread_id_);
  ASSERT(!shutdown_);
  // 首先在主线程执行任务
  cb();
  // 利用了shared_ptr自定义析构函数，在析构的时候向主线程post一个完成的通知任务
  // 这个机制和Bookkeeper的实现机制是一样的。
  std::shared_ptr<Event::PostCb> cb_guard(new Event::PostCb(cb),
                   [this, all_threads_complete_cb](Event::PostCb* cb) {
                    main_thread_dispatcher_->post(all_threads_complete_cb);
                      delete cb; });

  for (Event::Dispatcher& dispatcher : registered_threads_) {
    dispatcher.post([cb_guard]() -> void { (*cb_guard)(); });
  }
}
```

​	通过上面的代码可以看到，这里仍然利用到了`shared_ptr`的引用计数机制来实现的。每一个post到其他线程的任务都会导致`cb_guard`引用计数加1，post任务执行完成后`cb_guard`引用计数减1，等全部任务完成后，`cb_guard` 的引用计数就变成0了，这个时候就会执行自定义的删除器，在删除器中就会post一个任务到主线程中，从而实现了任务执行完成的通知回调机制。

​	接下来我们来分析下`shutdownGlobalThreading`，这个函数是用于设置flag来表示正在关闭TLS，必须由主线程在其它worker线程退出之前来调用，调用完成后每一个worker线程还需要调用对应TLS的`shutdownThread`来清理TLS中的对象，到此为止才完成了全部的TLS清理工作。

```c++
void InstanceImpl::shutdownGlobalThreading() {
  ASSERT(std::this_thread::get_id() == main_thread_id_);
  ASSERT(!shutdown_);
  shutdown_ = true;
}
```

上面的代码是`shutdownGlobalThreading`的实现，可以看到仅仅是设置了一个`shutdown_`的标志。


​	最后来分析一下`shutdownThread`，每一个work线程在退出事都需要调用这个函数，这个函数会将存储的所有线程存储的对象进行清除。每一个worker线程都持有`InstanceImpl`实例的引用，在析构的时候会调用`shutdownThread`来释放自己线程的TLS内容，这个函数的实现如下:

```cpp
void InstanceImpl::shutdownThread() {
  ASSERT(shutdown_);
  for (auto it = thread_local_data_.data_.rbegin(); 
	   it != thread_local_data_.data_.rend(); ++it) {
    it->reset();
  }
  thread_local_data_.data_.clear();
}
```

​	比较奇怪的点在于这里是逆序遍历所有的`ThreadLocalObject`对象来进行reset的，这是因为一些"持久"(活的比较长)的对象如`ClusterManagerImpl`很早就会创建`ThreadLocalObject`对象，但是直到shutdown的时候也不析构，而在此基础上依赖`ClusterManagerImpl`的对象的如`GrpcClientImpl`等，则是后创建`ThreadLocalObject`对象，如果`ClusterManagerImpl`创建的`ThreadLocalObject`对象先析构，而`GrpcClientImpl`相关的`ThreadLocalObject`对象依赖了`ClusterManagerImpl`相关的TLS内容，那么后析构就会导致未定义的问题。为此这里选择逆序来进行`reset`，先从一个高层的对象开始，最后才开始对一些基础的对象所关联的`ThreadLocalObject`进行`reset`。例如下面这个例子:

```cpp
struct ThreadLocalPool : public ThreadLocal::ThreadLocalObject {
	.....
  InstanceImpl& parent_;
  Event::Dispatcher& dispatcher_;
  Upstream::ThreadLocalCluster* cluster_;
	.....
};
```

​	`redis_proxy`中定义了一个`ThreadLocalPool`，这个`ThreadLocalPool`又依赖较为基础的`ThreadLocalCluster`(是`ThreadLocalClusterManagerImpl`的数据成员，也就是`ClusterManagerImpl`所对应的`ThreadLocalObject`对象)，如果`shutdownThread`按照顺序的方式析构的话，那么`ThreadLocalPool`中使用的`ThreadLocalCluster`会先被析构，然后才是`ThreadLocalPool`的析构，而`ThreadLocalPool`析构的时候又会使用到`ThreadLocalCluster`，但是`ThreadLocalCluster`已经析构了，这个时候就会出现野指针的问题了。

```cpp
ThreadLocalPool::ThreadLocalPool(InstanceImpl& parent, 
                                 Event::Dispatcher& dispatcher, const 
                                 std::string& cluster_name)
    : parent_(parent), dispatcher_(dispatcher), 
	cluster_(parent_.cm_.get(cluster_name)) {
  .....
  local_host_set_member_update_cb_handle_ = 
  cluster_->prioritySet().addMemberUpdateCb(
      [this](uint32_t, const std::vector<Upstream::HostSharedPtr>&,
             const std::vector<Upstream::HostSharedPtr>& hosts_removed) -> void {
        onHostsRemoved(hosts_removed);
      });
}

ThreadLocalPool::~ThreadLocalPool() {
  // local_host_set_member_update_cb_handle_是ThreadLocalCluster的一部分
  // ThreadLocalCluster析构会导致local_host_set_member_update_cb_handle_变成野指针
  local_host_set_member_update_cb_handle_->remove();
  while (!client_map_.empty()) {
    client_map_.begin()->second->redis_client_->close();
  }
}
```

​	到此为止关于Envoy中的TLS实现就全部分析完毕了。


# 小结

​	通过本节的分析相信我们应该足以驾驭Envoy中的`ThreadLocal`，从其设计可以看出它的一些其巧妙之处，比如抽象出一个`Slot`和对应的线程存储进行了关联，`Slot`可以任意传递，因为不包含实际的数据，拷贝的开销很低，只包含了一个索引值，具体关联的线程存储数据是不知道的，避免直接暴露给用户背后的数据。而`InstanceImpl`对象则管理着所有`Slot`的分配和移除以及整个`ThreadLocal`对象的`shutdown`。还有引入的Bookkeeper机制也甚是巧妙，和[Envoy源码分析之Dispatcher机制](https://envoyproxy-cn.github.io/blog/2020/08/23/envoy%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B9%8Bdispatcher%E6%9C%BA%E5%88%B6/)一文中的`DeferredDeletable`机制有着异曲同工之妙，通过这个机制可以做到安全的析构`SlotImpl`对象
