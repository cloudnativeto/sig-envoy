
---
title: "Envoy源码分析之Dispatcher机制"
linkTitle: "Envoy源码分析之Dispatcher机制"
date: 2020-08-23
description: Envoy和Nginx一样都是基于事件驱动的架构，这种架构的核心就是事件循环(EventLoop)。业界目前典型的几种事件循环实现主要有Libevent、Libev、Libuv、Boost.Asio等，也可以完全基于Linux系统调用epoll来实现。Envoy选择在Libevent的基础上进行了封装，实现了自己的事件循环机制，在Envoy中被称为`Dispatcher`，一个`Dispatcher`对象就是一个事件分发器，就如同它的名字一样。`Dispatcher`是Envoy的核心，可以说Envoy中绝大部分的能力都是构建在`Dispatcher`的基础上。所以理解`Dispatcher`机制是掌握Envoy的一个很重要的前提。
---

# Dispatcher机制

​	Envoy和Nginx一样都是基于事件驱动的架构，这种架构的核心就是事件循环(EventLoop)。业界目前典型的几种事件循环实现主要有Libevent、Libev、Libuv、Boost.Asio等，也可以完全基于Linux系统调用epoll来实现。Envoy选择在Libevent的基础上进行了封装，实现了自己的事件循环机制，在Envoy中被称为`Dispatcher`，一个`Dispatcher`对象就是一个事件分发器，就如同它的名字一样。`Dispatcher`是Envoy的核心，可以说Envoy中绝大部分的能力都是构建在`Dispatcher`的基础上。所以理解`Dispatcher`机制是掌握Envoy的一个很重要的前提。

​	在Envoy中`Dispatcher`不仅仅提供了网络事件分发、定时器、信号处理等基本的事件循环能力，还在事件循环的基础上实现任务执行队列、`DeferredDelet`等，这两个功能为Envoy中很多组件提供了必不可少的基础能力。比如借助`DeferredDelet`实现了安全的对象析构，通过任务执行队列实现Thread Local机制等等。

# Libevent事件封装

​	Envoy在Libevent的基础上进行了封装最为重要的一个原因就是因为Libevent本身是C开发的，很多Libevent暴露出来的结构需要自己来管理内存的分配和释放，这对于现代化的C++来说显然是无法接受的，因此Envoy借助了C++的RAII机制将这些结构封装起来，自动管理内存资源的释放。接下来我们看下Envoy是如何进行封装的。

```cpp
template <class T, void (*deleter)(T*)>
class CSmartPtr : public std::unique_ptr<T, void (*)(T*)> {
public:
  CSmartPtr() : std::unique_ptr<T, void (*)(T*)>(nullptr, deleter) {}
  CSmartPtr(T* object) : std::unique_ptr<T, void (*)(T*)>(object, deleter) {}
};
```

​	Envoy通过继承`unique_ptr`自定义了一个`CSmartPtr`，通过继承拥有了`unqiue_ptr`自动管理内存释放的能力，离开作用域后自动释放内存。借助`CSmartPtr`，Envoy将Libevent中的`event_base`包装成`BasePtr`，将`evconnlistener`包装成`ListenerPtr`。其中`event_base`就是事件循环，一个`event_base`就是一个事件循环，可以拥有多个事件循环，Envoy内部就是每一个worker线程都会有一个事件循环，也就是最常见的one loop per thread模型。

```cpp
using BasePtr = CSmartPtr<event_base, event_base_free>;
using ListenerPtr = CSmartPtr<evconnlistener, evconnlistener_free>;
```

​	在Libevent中无论是定时器到期、收到信号、还是文件可读写等都是事件，统一使用`event`类型来表示，Envoy中则将`event`作为`ImplBase`的成员，然后让所有的事件类型的对象都继承`ImplBase`，从而实现了事件的抽象。同时也借助了RAII机制自动实现了事件资源的释放。

```cpp
class ImplBase {
protected:
  ~ImplBase();
	
  event raw_event_;
};

ImplBase::~ImplBase() {
  // Derived classes are assumed to have already assigned the raw event in the constructor.
  event_del(&raw_event_);
}
```

​	 通过继承`ImplBase`基类可以拥有`event`事件成员，但是每一种事件表现出的具体行为是不一样的，比如说信号事件，需要有信号注册的能力，定时器事件则需要可以开启或者关闭定时的能力，文件事件则需要能够开启某些事件状态的监听。为此Envoy为每一种事件类型都抽象了对应的接口，例如文件事件接口。

```cpp
class FileEvent {
public:
  virtual ~FileEvent() = default;
  // 激活指定事件，会自动触发对应事件的callback
  virtual void activate(uint32_t events) PURE;
  // 开启指定事件状态的监听
  virtual void setEnabled(uint32_t events) PURE;
};
```

​	有了事件基类和对应的接口类后，让我们来看下Envoy如何来实现一个文件事件对象。

```cpp
// 通过继承ImplBase拥有了event成员
class FileEventImpl : public FileEvent, ImplBase {
public:
  FileEventImpl(DispatcherImpl& dispatcher, int fd, FileReadyCb cb, 
                FileTriggerType trigger,
                uint32_t events);

  // Event::FileEvent
  // 实现了文件事件的接口，通过这个接口可以实现文件事件的监听
  void activate(uint32_t events) override;
  void setEnabled(uint32_t events) override;

private:
  // 初始化事件对象
  void assignEvents(uint32_t events, event_base* base);
	
  // 事件触发时执行的callback
  FileReadyCb cb_;
  // 文件fd
  int fd_;
  // 事件触发的类型，边缘触发，还是水平触发
  FileTriggerType trigger_;
};

FileEventImpl::FileEventImpl(DispatcherImpl& dispatcher, int fd, FileReadyCb cb,
                             FileTriggerType trigger, uint32_t events)
    : cb_(cb), fd_(fd), trigger_(trigger) {
#ifdef WIN32
  RELEASE_ASSERT(trigger_ == FileTriggerType::Level,
                 "libevent does not support edge triggers on Windows");
#endif
  // dispatcher.base()返回的就是上文中说到的BasePtr，事件循环对象
  // 通过assignEvents初始化事件对象，设置好要监听的事件状态，以及事件回调callback等
  // 内部调用的就是Libevent的event_assign方法。
  assignEvents(events, &dispatcher.base());
  // 将事件对象注册到事件循环中，内部调用的就是
  event_add(&raw_event_, nullptr);
}
```

​	到此为止事件对象的封装就分析完了，接下来看下核心的`Dispatcher`对象，它提供了几个核心的方法来创建上文中分析的几个事件对象。

```cpp
  class DispatcherImpl {
   public:
     ....
     FileEventPtr createFileEvent(int fd, FileReadyCb cb, FileTriggerType trigger,
                                  uint32_t events) override;
     TimerPtr createTimer(TimerCb cb) override;
     SignalEventPtr listenForSignal(int signal_num, SignalCb cb) override;
     ....
  }
```

​	这就是`Dispatcher`对象的几个核心方法，在这几个方法的基础上又扩展了`createServerConnection`、`createClientConnection`等方法用于创建服务端和客户端连接对象，这两个方法内部最终都调用了`createFileEvent`方法，将socket文件的事件注册到了事件循环中。到此为止关于`Dispatcher`事件相关的几个方法都分析完了，但是`Dispatcher`对象远远还不止这些，比如说本文尚未提到的`Scheduler`，目前这个部分还尚未完成，这一块是对事件循环的抽象，目前是为了让事件循环组件可替换，目前只有`LibeventScheduler`一个实现。



# 任务执行队列

 	在上文中曾提到过Envoy在事件循环的基础上实现了两个比较重要的基础功能，其中一个就是任务执行队列了。可以随时通过`post`方法提交多个函数对象，然后交由`Dispatcher`来执行。所有的函数对象执行都是顺序的。是在`Dispatcher`所在的线程中执行。整个post方法的代码非常短。

```cpp
// 所有的要执行的函数对象原型都一样，都是void()
void DispatcherImpl::post(std::function<void()> callback) {
  bool do_post;
  {
    // 因为post方法可以跨线程执行，因此这里需要加锁来保证线程安全
    // 可以看出post方法本质上是将函数对象放到队列中，实际上并未执行
    Thread::LockGuard lock(post_lock_);
    do_post = post_callbacks_.empty();
    post_callbacks_.push_back(callback);
  }

  if (do_post) {
    post_timer_->enableTimer(std::chrono::milliseconds(0));
  }
}
```

​	`post`方法将传递进来的`callback`所代表的任务，添加到`post_callbacks_`所代表的类型为`vector<callback>`的成员变量中。如果`post_callbacks_`为空的话，说明背后的处理线程是处于非活动状态，这时通过`post_timer_`设置一个超时时间时间为0的方式来唤醒它。`post_timer_`在构造的时候就已经设置好对应的`callback`为`runPostCallbacks`，对应代码如下:

```cpp
DispatcherImpl::DispatcherImpl(TimeSystem& time_system,
							   Buffer::WatermarkFactoryPtr&& factory)
    : ......
      post_timer_(createTimer([this]() -> void { runPostCallbacks(); })),
      current_to_delete_(&to_delete_1_) {
  RELEASE_ASSERT(Libevent::Global::initialized(), "");
}
```

​	`runPostCallbacks`是一个while循环，每次都从`post_callbacks_`中取出一个`callback`所代表的任务去运行，直到`post_callbacks_`为空。每次运行`runPostCallbacks`都会确保所有的任务都执行完。显然，在`runPostCallbacks`被线程执行的期间如果`post`进来了新的任务，那么新任务直接追加到`post_callbacks_`尾部即可，而无需做唤醒线程这一动作。

```cpp
void DispatcherImpl::runPostCallbacks() {
  while (true) {
    std::function<void()> callback;
    {
      Thread::LockGuard lock(post_lock_);
      if (post_callbacks_.empty()) {
        return;
      }
      callback = post_callbacks_.front();
      post_callbacks_.pop_front();
    }
    callback();
  }
}
```

​	到此为止Envoy中的任务执行队列就分析完了，可以看出这个部分的代码实现还是很简单的，也很容易验证其正确性，在Envoy的代码中被广泛使用。这个能力和Boost::asio中的post task是类似的。


# DeferredDeletable

​	本小节是`Dispatcher`中最重要的一个部分`DeferredDeletable`，又被称为延迟析构，目的是用于安全的进行对象析构。C++语言本身会存在对象析构了，但还有引用它的指针存在，这个时候通过这个指针访问这个对象就会导致未定义行为了。因此写C++的同学就需要特别注意一个对象的生命周期问题，要保证引用一个对象的时候，对象还没有被析构。在C++中有不少方案可以来解决这个问题，典型的像使用shared_ptr的方式。而本文的要分析的`DeferredDeletable`则是使用另外一种方式来解决对象安全析构问题，这个方案的并不是一个通用的方案，仅能解决部分场景下的对象安全析构问题，但是对于Envoy使用到的场景已经足够了，接下来我们将分析它是如何做到对象安全析构的。

​	`DeferredDeletable`本身是一个空接口，所有要进行延迟析构的对象都要继承自这个空接口。在Envoy的代码中像下面这样继承自`DeferredDeletable`的类随处可见。

```cpp
class DeferredDeletable {
public:
  virtual ~DeferredDeletable() {}
};

class Connection : public Event::DeferredDeletable { .... }

/**
 * An instance of a generic connection pool.
 */
class Instance : public Event::DeferredDeletable { ..... }

/**
 * Implementation of AsyncRequest. This implementation is capable of 
 * sending HTTP requests to a ConnectionPool asynchronously.
 */
class AsyncStreamImpl : public Event::DeferredDeletable{....}
```

​		这些继承`DeferredDeletable`接口的类都有一个特点，这些类基本上都是一些具有短暂生命周期的对象，比如连接对象、请求对象等。这也正是上文中提到的延迟析构并非是是一个通用方案，只是针对Envoy中的一些特定场景。`DeferredDeletable`和`Dispatcher`是密切相关，是基于`Dispatcher`来完成的。`Dispatcher`对象有一个`vector`保存了所有要延迟析构的对象。

```cpp
class DispatcherImpl : public Dispatcher {
  ......
 private:
  ........
  std::vector<DeferredDeletablePtr> to_delete_1_;
  std::vector<DeferredDeletablePtr> to_delete_2_;
  std::vector<DeferredDeletablePtr>* current_to_delete_;
 }
```

​	`to_delete_1_`和`to_delete_2_`就是用来存放所有的要延迟析构的对象，这里使用两个`vector`存放，为什么要这样做呢？或许可能有人会想这是因为要保证线程安全，不能往一个正在析构的列表中添加对象。其实并非如此，多线程操作一个队列本就是非线程安全的，所以这里使用两个队列的目的并非是为了线程安全的。带着这个疑问继续往下分析，`current_to_delete_`始终指向当前正要析构的对象列表，每次执行完析构后就交替指向另外一个对象列表，来回交替。

```cpp
void DispatcherImpl::clearDeferredDeleteList() {
  ASSERT(isThreadSafe());
  std::vector<DeferredDeletablePtr>* to_delete = current_to_delete_;
  size_t num_to_delete = to_delete->size();
  // 如果正在删除或者没有对象可删除就返回
  if (deferred_deleting_ || !num_to_delete) {
    return;
  }
  // 正式开始删除对象
  ENVOY_LOG(trace, "clearing deferred deletion list (size={})", num_to_delete);
  // current_to_delete_指向另外一个没有进行删除的队列
  if (current_to_delete_ == &to_delete_1_) {
    current_to_delete_ = &to_delete_2_;
  } else {
    current_to_delete_ = &to_delete_1_;
  }
  // 设置正在删除的标志
  deferred_deleting_ = true;
  // 开始进行对象析构
  for (size_t i = 0; i < num_to_delete; i++) {
    (*to_delete)[i].reset();
  }
	
  to_delete->clear();
  // 结束
  deferred_deleting_ = false;
}
```

​	上面的代码中我们可以看到在执行对象析构的时候先使用`to_delete`来指向当前正要析构的对象列表，然后将`current_to_delete_`指向另外一个列表，这里为什么要设置`deferred_deleting_`标志呢? 这是因为`clearDeferredDeleteList`可能会被调用多次，如果已经有对象正在析构，那么就不能再进行析构操作了，因此这里通过`deferred_deleting_`标志来保证同一时刻只能有一个对象析构的任务在执行。



> 假设没有`deferred_deleting_`标志，如果此时正在执行`to_delete_1_`队列的对象析构，在析构的过程中调用了`clearDeferredDeleteList`，那么这个时候会对`to_delete_2_`队列开始析构，并且将`current_to_delete_`指向`to_delete_1_`，后续的待析构对象就都会添加到`to_delete_1_`队列中，这可能会导致对`to_delete_1_`析构的任务执行较长时间。影响其它关键任务的执行。



​	接下来我们来看下如何将对象添加到待析构的列表中。

```cpp
void DispatcherImpl::deferredDelete(DeferredDeletablePtr&& to_delete) {
  ASSERT(isThreadSafe());
  current_to_delete_->emplace_back(std::move(to_delete));
  ENVOY_LOG(trace, "item added to deferred deletion list (size={})", current_to_delete_->size());
  if (1 == current_to_delete_->size()) {
    deferred_delete_timer_->enableTimer(std::chrono::milliseconds(0));
  }
}
```

​	`deferredDelete`和`clearDeferredDeleteList`这两个方法都调用了` ASSERT(isThreadSafe());`目的是断言调用这两个方法是在Dispatcher所在线程执行的，是单线程运行。可以保证线程安全。 既然如此我们便可以安全的往待析构的对象列表中追加对象了，这也验证了两个队列的设计并非是为了线程安全。那为何还要搞出`to_delete_1_`和`to_delete_2_`两个列表呢?  完全可以通过一个列表来实现，通过while循环不断的进行对象析构，直到列表为空。在处理的过程中还可以往列表中追加对象。

```cpp
while(!current_to_delete_.empty()) {
	auto obj = current_to_delete_.pop_back();
	//  进行业务逻辑的处理
}
```

​	从功能正确性的角度来看，这里使用两个列表，还是一个列表都可以正确实现，在上文中分析的任务执行队列其实就是使用一个列表来完成的。但是Envoy在这里选择了两个队列的方式，这是因为相比于任务执行队列来说延迟析构的重要性更低一些，大量对象的析构如果保存在一个队列中循环的进行析构势必会影响其他关键任务的执行，所以这里拆分成两个队列，多个任务交替的执行，避免被一个大的耗时任务长期占用，导致其他关键任务无法及时执行。

> 如果用一个队列做对象析构，在对象的析构函数中可能还会再次调用deferredDelete将新的对象追加到待析构的列表中，所以可能会导致队列中的任务不断增加，造成整个对象析构耗时较长。



​	继续看`deferredDelete`的代码我们会发现另外一个问题，为何要在当前待析构对象的列表大小等于1的时候唤起定时器任务呢?

```cpp
if (1 == current_to_delete_->size()) {
  // deferred_delete_timer_(createTimerInternal([this]() -> void { 			  
  // clearDeferredDeleteList(); })),
  // deferred_delete_timer_定时器对应的任务就是clearDeferredDeleteList
  deferred_delete_timer_->enableTimer(std::chrono::milliseconds(0));
}
```

​	假设我们每次添加对象到当前列表中都进行唤醒，那么带来的问题就是`clearDeferredDeleteList`的任务会有多个，但是实际上只有两个队列，只需要有两个`clearDeferredDeleteList`任务就可以将两个队列中的对象都析构掉，那么剩下的任务将不会进行任何实际的工作。很显然这样会带来CPU上的浪费，因此我们应该尽可能的少唤醒，保证任何时候最多只有两个任务。因此我们只要能保证在每次队列为空的时候唤醒一次即可，因为唤醒的这次任务会负责将这个队列变为空，到时候在此唤醒一个任务即可。这也就是为什么这里通过判断当前待析构对象的列表大小等于1的原因了。

​	到此为止`deferredDelete`的实现原理就基本分析完了，可以看出它的实现和任务队列的实现很类似，只不过一个是循环执行`callback`所代表的任务，另一个是让对象进行析构。最后让我们通过下图来看下整个`deferredDelete`的流程。

![4-1.png](https://ata2-img.oss-cn-zhangjiakou.aliyuncs.com/ebb781960da723965cdd9d968dc10258.png)

* 对象要被析构了，开始调用`deferredDelete`将对象添加到`to_delete_1`队列中，然后唤醒`clearDeferredDeleteList`任务。
* `clearDeferredDeleteList`任务开始执行，`current_to_delete`指向`to_delete_2`队列
* 对象在析构的过程中又通过`deferredDelete`添加了新的对象到to_delete_2队列中，这个队列初始是空的，因此再次唤醒一个`clearDeferredDeleteList`任务。
* `to_delete_1`队列继续进行对象的析构，在析构期间有大量对象被添加到`to_delete_2`队列中，但是没有唤醒`clearDeferredDeleteList`任务。
* to_delete_1对象析构完毕
* 再次执行`clearDeferredDeleteList`对`to_delete_2`中对象进行析构。
* 如此反复便可以高效的在两个队列之间来回切换进行对象的析构。



​	虽然分析完了整个`deferredDelete`的过程，但是我们还没有回答本节一开始提到的如何安全的进行对象析构的问题。让我们先来看一下`deferredDelete`的应用场景，看看“为何要进行延迟析构?” 以及`deferredDelete`是如何解决对象安全析构的问题。在Envoy的源代码中经常会看到像下面这样的代码片段。

```cpp
ConnectionImpl::ConnectionImpl(Event::Dispatcher& dispatcher, 
							   ConnectionSocketPtr&& socket,
                               TransportSocketPtr&& transport_socket,
							   bool connected) {
......
  }
  // 传递裸指针到回调中
  file_event_ = dispatcher_.createFileEvent(
    	// 这里将this裸指针传递给了内部的callback
      // callback内部通过this指针访问onFileEvent方法，如何保证
      // callback执行的时候，this指针是有效的呢?
      fd(), [this](uint32_t events) -> void { onFileEvent(events); }, 
      Event::FileTriggerType::Edge,
      Event::FileReadyType::Read | Event::FileReadyType::Write);
	......
}
```

​	传递给`Dispatcher`的`callback`都是通过裸指针的方式进行回调，如果进行回调的时候对象已经析构了，就会出现野指针的问题，我相信学过C++的同学都会看出这个问题，除非能在逻辑上保证`Dispatcher`的生命周期比所有对象都短，这样就能保证在回调的时候对象肯定不会析构，但是这不可能成立的，因为`Dispatcher`是`EventLoop`的核心。一个线程运行一个`EventLoop`直到线程结束，`Dispatcher`对象才会析构，这意味着`Dispatcher`对象的生命周期是最长的。所以从逻辑上没办法保证进行回调的时候对象没有析构。可能有人会有疑问，对象在析构的时候把注册的事件(`file_event_`)取消不就可以避免野指针的问题吗? 那如果事件已经触发了，`callback`正在等待运行？ 又或者`callback`运行了一半呢？前者libevent是可以保证的，在调用`event_del`删除事件的时候可以把处于等待运行的事件callback取消掉，但是后者就无能为力了，这个时候如果对象析构了，那行为就是未定义了。沿着这个思路想一想，是不是只要保证对象析构的时候没有`callback`正在运行就可以解决问题了呢？是的，只要保证所有在执行中的`callback`执行完了，再做对象析构就可以了。可以利用`Dispatcher`是顺序执行所有`callback`的特点，向`Dispatcher`中插入一个任务就是用来对象析构的，那么当这个任务执行的时候是可以保证没有其他任何`callback`在运行。通过这个方法就完美解决了这里遇到的野指针问题了。或许有人又会想，这里是不是可以用[shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)和[shared_from_this](https://en.cppreference.com/w/cpp/memory/enable_shared_from_this/shared_from_this)来解这个呢? 是的，这是解决多线程环境下对象析构的秘密武器，通过延长对象的生命周期，把对象的生命周期延长到和`callback`一样，等`callback`执行完再进行析构，同样可以达到效果，但是这带来了两个问题，第一就是对象生命周期被无限拉长，虽然延迟析构也拉长了生命周期，但是时间是可预期的，一旦`EventLoop`执行了`clearDeferredDeleteList`任务就会立刻被回收，而通过`shared_ptr`的方式其生命周期取决于`callback`何时运行，而`callback`何时运行这个是没办法保证的，比如一个等待`socket`的可读事件进行回调，如果对端一直不发送数据，那么`callback`就一直不会被运行，对象就一直无法被析构，长时间累积会导致内存使用率上涨。第二就是在使用方式上侵入性较强，需要强制使用`shared_ptr`的方式创建对象。
