---
title: "Librbd学习2: 写I/O流程"
date: 2022-07-28T08:22:07Z
tags: ["librbd"]
draft: false
---


## 写I/O处理流程图

先把自己根据I/O栈总结的流程图放上来，后续的分析以该图为基准，顺着这个图来查看具体流程能够加深理解:)

![](D:\work\学习笔记\ceph学习\librbd工作原理\write\src\write_IO_flow.png)

图中不同的阶段有不同的颜色，其中：

* 黄色部分，代表客户端程序主线程把写操作追加到工作队列的步骤
* 蓝色部分，为Image内置线程池中的worker线程从工作队列中取出一次(写)请求的步骤
* 绿色部分，为该写请求的处理在librbd层上的步骤
* 橙色部分，为该写请求的处理在librados层上的步骤
* 粉色部分，为osdc层中的处理步骤
* 灰色部分，为msg层中的处理，此次I/O流程图未展开



## 客户端侧写入流程

### 主要工作内容

* Image.write实际上就是把相关请求的参数，丢入到image成员变量ctx维护的一个ImageRequestWQ类型的队列中。
* ImageRequestWQ处理的主要工作：
  * 对I/O进行了参数检查和裁剪
  * 使用异步I/O接口(aio_write)实现了同步I/O(write)的功能
  * 异步I/O(aio_write)中，把write请求打包为ImageDispatchSpec*类型的req，并调用PointerWQ.queue进行入列
* PointerWQ的queue方法负责把item(req)追加到std::list类型的成员变量m_items的尾部，主要流程到此结束

### 调用librbd::Image中write方法

```c++
Image::write()
  ImageCtx *ictx = (ImageCtx *)ctx;
  // 调用imageContext的io_work_queue的write, io_work_queue是一个ImageRequestWQ类型。
  return ictx->io_work_queue->write(ofs, len, bufferlist{bl}, 0); -> ImageRequestWQ<I>::write(); 
```

### ImageRequestWQ中的流程

实际调用了ImageRequestWQ的aio_write方法，通过等待回调的方式实现同步效果

```c++
ImageRequestWQ<I>::write()
  m_queued_writes++;
  ThreadPool::PointerWQ<ImageDispatchSpec<I> >::queue(req); // 调用ThreadPool::PointerWQ的queue方法ImageRequestWQ<I>::write
  int r = clip_io(util::get_image_ctx(&m_image_ctx), off, &len); // 进行参数检查，并且当len越界时，裁剪len至size的最大有效值
  C_SaferCond cond;
  AioCompletion *c = AioCompletion::create(&cond); // 构建异步IO完成时的回调
  aio_write(c, off, len, std::move(bl), op_flags, false); -> ImageRequestWQ<I>::aio_write() // 调用aio_write异步写接口
  r = cond.wait(); // 等待I/O完成，实现同步I/O效果

ImageRequestWQ<I>::aio_write()
  // 创建一个write请求
  ImageDispatchSpec<I> *req = ImageDispatchSpec<I>::create_write_request(
          m_image_ctx, c, {{off, len}}, std::move(bl), op_flags, trace, tid);
  queue(req); -> ImageRequestWQ<I>::queue() // 把write请求丢入queue中

ImageRequestWQ<I>::queue()
  m_queued_writes++;
  ThreadPool::PointerWQ<ImageDispatchSpec<I> >::queue(req); // 显式调用ImageRequestWQ的基类ThreadPool::PointerWQ的queue方法
```

### 调用PointerWQ的queue方法

```c++
ThreadPool::PointerWQ<ImageDispatchSpec<I> >::queue()
  std::lock_guard l(m_pool->_lock); // 对整个ThreadPool添加一个排他锁
  m_items.push_back(item); // std::list类型的m_items后部追加一个item，即流程图中index为1的元素
  m_pool->_cond.notify_one();
```

使用GDB查看m_items列表在添加item前后状态

```
(gdb) n
378           m_items.push_back(item);
(gdb) p m_items.size()
$10 = 1
(gdb) n
379           m_pool->_cond.notify_one();
(gdb) p m_items.size()
$11 = 2
(gdb) p m_items.back()
$12 = (Context *) 0x7fffb400ee50
```

在执行push_back之后，m_items的长度增加1，新增了一个Context*指针类型的元素。至此，客户端主线程的核心工作已经完成。



## 工作线程处理请求

线程池中的worker线程从队列中拉取请求并执行

### 主要工作内容

* ThreadPool::worker实现了线程池中每一个工作线程的业务流程，并在两个循环中展开
* 外层循环确保该线程一直在工作，直到被标记为_stop
* 内存循环通过一个公共的index，每次遍历work_queues得到其中某个work_queue，并出列一个item(ImageDispatchSpec*类型)
* 如果该item不为空，则使用work_queue的_void_process方法处理该item

### ThreadPool的定义

```c++
class ThreadPool : public md_config_obs_t
{
protected:
    CephContext *cct;
    std::string name;
    std::string thread_name;
    std::string lockname;
    ceph::mutex _lock;
    ceph::condition_variable _cond;
    bool _stop; // 控制是否停止的状态值
    int _pause; // 控制是否暂停
    int _draining;
    ceph::condition_variable _wait_cond;

protected:
    // 维护的多个WorkQueue或者其子类，比如WorkQueueVal, BatchWorkQueue, PointerWQ, ImageRequestWQ等
    std::vector<WorkQueue_ *> work_queues;
    int next_work_queue = 0;
}
```

### 执行worker方法

```c++
void ThreadPool::worker(WorkThread *wt)
{
    ...
    while (!_stop) // 循环执行，除非_stop被置为false
    {
        ...
        if (!_pause && !work_queues.empty()) // 当_pause为0时，且work_queues不为空时执行
        {
            int tries = 2 * work_queues.size(); // 最大重试次数
            while (tries--) // 在最大重试次数内，循环执行
            {
                ...
                next_work_queue %= work_queues.size();
                wq = work_queues[next_work_queue++]; // 获取下一个work_queue

                void *item = wq->_void_dequeue(); // 出列work_queue中的item
                if (item) // 如果item存在，则处理该item
                {
                    ...
                    wq->_void_process(item, tp_handle); // 执行_void_process对item进一步处理
                    ...
                    break;
                }
            }
        }
    }
}
```

### 执行work_queue的_void_dequeue方法

在我们写请求的场景中，需要关心的work_queue的类型应该为ImageRequestWQ*，_void_dequeue的实现在其父类PointerWQ中，

```c++
template <typename T>
class PointerWQ : public WorkQueue_
{
protected:
    ...
    void *_void_dequeue() override
    {
        if (m_items.empty()) // 列表为空直接返回
        {
            return NULL;
        }

        ++m_processing;
        T *item = m_items.front(); // 获取队列头部元素，即流程图中index为0的元素
        m_items.pop_front(); // 弹出队列头
        return item;
    }
    ...
}
```



## 写请求在librbd层的处理

### 主要工作内容

* 利用ImageRequestWQ工作队列，对某请求执行阻塞控制，对读写操作进行保序
* 在ImageWriteRequest的处理下，结合块的layout信息，把对块内地址范围(image_extent)的请求，拆分成对块内多个对象的地址范围(object_extent)的请求
* 在ObjectWriteRequest中，把object_extent转换成librados::ObjectWriteOperation，其内部包含了更下层osdc(OSD客户端)层能够使用的OSDOp

### ImageRequestWQ中的处理

ThreadPool::worker中的wq._void_process在基类PointerWQ中实现，

```c++
template <typename T>
class PointerWQ : public WorkQueue_
{
protected:
    ...
    void _void_process(void *item, ThreadPool::TPHandle &handle) override
    {
        process(reinterpret_cast<T *>(item));
    }
    ...
}
```

其内调用的process在子类ImageRequestWQ*中被覆盖实现，

```c++
template <typename I>
void ImageRequestWQ<I>::process(ImageDispatchSpec<I> *req) {
  ...
  process_io(req, true); // 调用process_io继续处理
  ...
}
```

ImageRequestWQ.process_io()方法

主要判断该req是否能够被立即执行，如果不能立即执行，需要重新入列继续排队等待。

```C++
template <typename I>
void ImageRequestWQ<I>::process_io(ImageDispatchSpec<I> *req,
                                   bool non_blocking_io) {
  ...
  if (write_op && !req->blocked) {
    bool blocked = block_overlapping_io(&m_in_flight_extents, offset, length); // 判断该io是否该被阻塞
    if (blocked) { // 如果阻塞了
	  ...
      req->blocked = true; // 设置blocked标志位
      m_blocked_ios.push_back(req); // 重新丢入m_blocked_ios队列中
      return;
    }
  }
  ...
  req->send(); // 调用ImageDispatchSpec的send方法
  ...
  delete req;
}
```

ImageRequestWQ.block_overlapping_io()方法

其主要是为了避免写I/O的冲突，即保证同一个I/O地址范围内的写时序，防止类似lost update等冲突问题，实现上比较简单

```C++
template <typename I>
bool ImageRequestWQ<I>::block_overlapping_io(
    ImageExtentIntervals* in_flight_image_extents, uint64_t off, uint64_t len) {
  if(len == 0) { // 写I/O的数据长度为0，不会更改数据，故不会阻塞
    return false; // 返回不堵塞
  }

  if (in_flight_image_extents->empty() || // 如果当前没有在途的I/O，则不堵塞
      !in_flight_image_extents->intersects(off, len)) { // 或者在途的I/O中没有跟此次写入I/O地址重叠的，则不堵塞
    in_flight_image_extents->insert(off, len); // 把此次I/O放入in_flight_image_extents集合中
    return false; // 返回不堵塞
  }

  return true;
}
```

主要工作内容：

* worker线程从工作队列(WQ)中取出一个写请求之后，需要检查该请求是否需要被阻塞
* 判断阻塞的主要标准是，该请求所需要写入的extents是否与工作队列上在途的extents(in_flight_image_extents)有重叠

### ImageDispatchSpec中的处理

ImageDispatchSpec是一个中间调度层，主要的职责是把从ImageRequestWQ取出的req(即ImageDispatchSpec*对象本身)，分发到具体的ImageRequest上继续执行。

进一步下一步前，需要再具体回顾一下，客户端主线程ImageRequestWQ::aio_write()中，调用ImageDispatchSpec::create_write_request创建req的内容

```c++
static ImageDispatchSpec* create_write_request(
    ImageCtxT &image_ctx, AioCompletion *aio_comp, Extents &&image_extents,
    bufferlist &&bl, int op_flags, const ZTracer::Trace &parent_trace, uint64_t tid) {
  return new ImageDispatchSpec(image_ctx, aio_comp, std::move(image_extents),
                               Write{std::move(bl)}, op_flags, parent_trace, tid); // 使用bl构造了个Write结构体类型的对象
}

// 构造函数
ImageDispatchSpec(ImageCtxT& image_ctx, AioCompletion* aio_comp,
                   Extents&& image_extents, Request&& request,
                   int op_flags, const ZTracer::Trace& parent_trace, uint64_t tid)
  : m_image_ctx(image_ctx), m_aio_comp(aio_comp),
    m_image_extents(std::move(image_extents)), m_request(std::move(request)), // 使用request初始化m_request(apply_visitor中会用到)
    m_op_flags(op_flags), m_parent_trace(parent_trace), m_tid(tid) {
  m_aio_comp->get();
}

// Request类型定义
typedef boost::variant<Read,
                       Discard,
                       Write,
                       WriteSame,
                       CompareAndWrite,
                       Flush> Request; // 包含6种类型的boot可变类型
```

ImageDispatchSpec.send()方法

实现比较简单，直接调用了boost::apply_visitor函数。

```
template <typename I>
void ImageDispatchSpec<I>::send() {
  boost::apply_visitor(SendVisitor{this}, m_request);
}
```

根据上面的分析，这里的m_request是类型为可变类型Request的引用，而被引用者是一个Write对象。

在SendVisitor中，会选择匹配的类型执行operator()函数：

```C++
struct ImageDispatchSpec<I>::SendVisitor
  : public boost::static_visitor<void> {
  void operator()(Read& read) const {
    ...
  }

  void operator()(Discard& discard) const {
    ...
  }

  void operator()(Write& write) const { // 动态地根据m_request的实际类型Write，调用到此处
    ImageRequest<I>::aio_write( // 调用aio_write方法
      &spec->m_image_ctx, spec->m_aio_comp, std::move(spec->m_image_extents),
      std::move(write.bl), spec->m_op_flags, spec->m_parent_trace);
  }

  void operator()(WriteSame& write_same) const {
    ...
  }
  ...
}
```

ImageRequest.aio_write()函数

```c++
template <typename I>
void ImageRequest<I>::aio_write(I *ictx, AioCompletion *c,
                                Extents &&image_extents, bufferlist &&bl,
                                int op_flags,
				                const ZTracer::Trace &parent_trace) {
  // 创建一个ImageWriteRequest
  ImageWriteRequest<I> req(*ictx, c, std::move(image_extents), std::move(bl),
                           op_flags, parent_trace);
  req.send(); // 执行其send方法
}
```

主要工作内容:

* ImageDispatchSpec的主要作用，是根据其在构造时的m_request(boost::variant类型)，动态调度到对应的ImageRequest上，执行request。



### ImageWriteRequest中的处理

此处涉及到的几个类及他们之间的关系

![](.\src\ImageRequestClass.drawio.png)

ImageWriteRequest.send()方法，实现于基类ImageRequest中

```C++
template <typename I>
void ImageRequest<I>::send() {
  ...
  int r = clip_request(); // 裁剪请求
  if (r < 0) {
    m_aio_comp->fail(r);
    return;
  }

  if (finish_request_early()) { // 根据m_image_extents计算total_bytes, 如果total_bytes == 0, 返回true
    return; // 当待写入的total_bytes为0时, 直接返回, 提前结束请求
  }

  if (m_bypass_image_cache || m_image_ctx.image_cache == nullptr) { // 如果置了m_bypass_image_cache, 或m_image_ctx.image_cache为空
    update_timestamp();
    send_request();  // 发送该请求
  } else {
    send_image_cache_request();
  }
}
```

ImageWriteRequest.send_request()方法，实现于父类AbstractImageWriteRequest中

```c++
template <typename I>
void AbstractImageWriteRequest<I>::send_request() {
  ...
  LightweightObjectExtents object_extents; // 创建对象的extents, 实际类型为boost::container::small_vector的向量
  for (auto &extent : this->m_image_extents) { // 遍历image的多个地址空间分片(extents)的每一个
    if (extent.second == 0) { // 如果该image_extent的len为0, 对于写操作来说是无效的，直接丢弃
      continue;
    }
    ...
    Striper::file_to_extents(cct, &image_ctx.layout, extent.first, // 把image的地址空间分片，转换为多个对象的地址空间分片
                             extent.second, 0, clip_len, &object_extents);
    ...
  }
  ...
  if (!object_extents.empty()) { // 如果向量不为空
    ...
    send_object_requests(object_extents, snapc, journal_tid); // 调用到send_object_requests
  }
  ...
}
```

Striper::file_to_extents()函数

对于一个Image地址空间的写请求的extent，该函数可以把它转换成多个对象地址空间上的写请求的extent。

除了image_extent.first(offset)和image_extent.second(length)作为输入外，还需要传入image_ctx.layout。layout中保存了当前image中比较重要的几个元数据，在rbd.open(image)之后会读取下来，其定义如下：

```C++
struct file_layout_t {
  // file -> object mapping
  uint32_t stripe_unit;   ///< stripe unit, in bytes,
  uint32_t stripe_count;  ///< over this many objects
  uint32_t object_size;   ///< until objects are this big

  int64_t pool_id;        ///< rados pool id
  std::string pool_ns;         ///< rados pool namespace
}
```

函数的内容相对较为复杂，因为不是此次讨论的重点，下面给出原理图方便理解。

原理图(转载自陈小跑，https://www.cnblogs.com/chenxianpao/p/5572859.html)

![](.\src\file_to_extents.jpg)



ImageWriteRequest.send_request()方法，实现于父类AbstractImageWriteRequest中

```c++
template <typename I>
void AbstractImageWriteRequest<I>::send_object_requests(
    const LightweightObjectExtents &object_extents, const ::SnapContext &snapc,
    uint64_t journal_tid) {
  ...
  for (auto& oe : object_extents) { // 遍历每一个对象的地址空间分片
    ...
    auto request = create_object_request(oe, snapc, journal_tid, single_extent, req_comp); // 创建一个对象的写请求
    request->send(); // 执行请求的send方法
  }
}
```

ImageWriteRequest.create_object_request()方法

```C++
template <typename I>
ObjectDispatchSpec *ImageWriteRequest<I>::create_object_request(
    const LightweightObjectExtent &object_extent, const ::SnapContext &snapc,
    uint64_t journal_tid, bool single_extent, Context *on_finish) {
  I &image_ctx = this->m_image_ctx;

  bufferlist bl;
  if (single_extent && object_extent.buffer_extents.size() == 1 && // 该object_extent仅包含一个连续的地址段
      m_bl.length() == object_extent.length) {
    // optimization for single object/buffer extent writes
    bl = std::move(m_bl); // 直接使用image请求的buffer_list构造object请求的buffer_list
  } else {
    assemble_extent(object_extent, &bl); // 根据object_extent中定义的多个地址段，从image请求的bl中截取出object_extent范围内的bl
  }

  auto req = ObjectDispatchSpec::create_write( // 创建一个ObjectDispatchSpec类型的req
    &image_ctx, OBJECT_DISPATCH_LAYER_NONE, object_extent.object_no,
    object_extent.offset, std::move(bl), snapc, m_op_flags, journal_tid,
    this->m_trace, on_finish);
  return req;
}
```

主要工作内容:

* ImageWriteRequest首先做了I/O裁剪、是否能够直接完求、是否需要缓存处理等工作
* 然后使用file_to_extents函数，把Image视角的写请求地址空间分片(image_extents)，转换成为一组对象视角的写请求地址空间分片(object_extent)
* 再遍历这组对象地址分片，创建多个对象请求分发说明(ObjectDispatchSpec)，过程中还会根据extent的范围，重新组装对象请求自己的bufferlist
* 最终执行send方法，执行对象请求分发说明中的相关逻辑

### ObjectDispatchSpec中的处理

与ImageDispatchSpec原类似，ObjectDispatchSpec.send()方法，最终会调用到ObjectDispatcher::SendVisitor::operator(WriteRequest& write)中。

```C++
// 可变类型Request定义
typedef boost::variant<ReadRequest,
                       DiscardRequest,
                       WriteRequest,
                       WriteSameRequest,
                       CompareAndWriteRequest,
                       FlushRequest> Request;

template <typename ImageCtxT>
static ObjectDispatchSpec* create_write(
    ImageCtxT* image_ctx, ObjectDispatchLayer object_dispatch_layer,
    uint64_t object_no, uint64_t object_off, ceph::bufferlist&& data,
    const ::SnapContext &snapc, int op_flags, uint64_t journal_tid,
    const ZTracer::Trace &parent_trace, Context *on_finish) {
  return new ObjectDispatchSpec(image_ctx->io_object_dispatcher,
                                object_dispatch_layer,
                                WriteRequest{object_no, object_off, // 使用WriteRequest填充ObjectDispatchSpec的request属性
                                             std::move(data), snapc,
                                             journal_tid},
                                op_flags, parent_trace, on_finish);
}

// 最终动态调度到拥有WriteRequest参数的operator(根据不同的Request类型进行了重载)方法上
bool operator()(ObjectDispatchSpec::WriteRequest& write) const {
  return object_dispatch->write(
    write.object_no, write.object_off, std::move(write.data), write.snapc,
    object_dispatch_spec->op_flags, object_dispatch_spec->parent_trace,
    &object_dispatch_spec->object_dispatch_flags, &write.journal_tid,
    &object_dispatch_spec->dispatch_result,
    &object_dispatch_spec->dispatcher_ctx.on_finish,
    &object_dispatch_spec->dispatcher_ctx);
}
```

ObjectDispatch.write()方法

```C++
template <typename I>
bool ObjectDispatch<I>::write(
    uint64_t object_no, uint64_t object_off, ceph::bufferlist&& data,
    const ::SnapContext &snapc, int op_flags,
    const ZTracer::Trace &parent_trace, int* object_dispatch_flags,
    uint64_t* journal_tid, DispatchResult* dispatch_result,
    Context** on_finish, Context* on_dispatched) {
  ...
  auto req = new ObjectWriteRequest<I>(m_image_ctx, object_no, object_off, // 创建一个ObjectWriteRequest请求对象
                                       std::move(data), snapc, op_flags,
                                       parent_trace, on_dispatched);
  req->send(); // 执行ObjectWriteRequest请求的send()方法
  return true;
}
```

主要工作内容:

* ObjectDispatchSpec中，主要是根据创建它时指定的实际Request类型，创建不同的ObjectRequest进行处理。

### ObjectWriteRequest中的处理

当前场景下主要使用了两个类，ObjectWriteRequest与其父类AbstractObjectWriteRequest，他们之前的关系如下：

![](.\src\ObjectRequestClass.drawio.png)

AbstractObjectWriteRequest.send()方法

```c++
template <typename I>
void AbstractObjectWriteRequest<I>::send() {
  ...
  {
    ...
    if (image_ctx->object_map == nullptr) {
      m_object_may_exist = true;
    } else {
      ...
      m_object_may_exist = image_ctx->object_map->object_may_exist(this->m_object_no); // 根据对象号到object_map查询该对象是否存在
    }
  }

  if (!m_object_may_exist && is_no_op_for_nonexistent_object()) { // 如果对象不存在，且该请求不包含对象操作(op)
    ldout(image_ctx->cct, 20) << "skipping no-op on nonexistent object"
                              << dendl;
    this->async_finish(0); // 直接完成
    return;
  }

  pre_write_object_map_update();
}
```

AbstractObjectWriteRequest.pre_write_object_map_update()方法

主要更新用于保存镜像与父镜像之间映射关系的object_map

```c++
template <typename I>
void AbstractObjectWriteRequest<I>::pre_write_object_map_update() {
  I *image_ctx = this->m_ictx;

  image_ctx->image_lock.lock_shared();
  if (image_ctx->object_map == nullptr || !is_object_map_update_enabled()) { // 如果镜像的object_map不存在
    image_ctx->image_lock.unlock_shared();
    write_object();
    return;
  }

  if (!m_object_may_exist && m_copyup_enabled) { // 如果对象不存在
    // optimization: copyup required
    image_ctx->image_lock.unlock_shared();
    copyup(); // 处理克隆镜像的场景，先创建一个CopyupRequest请求把数据拷贝下来，再执行写对象请求
    return;
  }
  ...
  write_object();
}
```

AbstractObjectWriteRequest.write_object()方法

主要功能是把针对某对象的写的请求(ObjectWriteRequest)，转换成针对OSD的ObjectWriteOperation，并向其填充ObjectOperation，最后使用data_ctx执行。

```C++
template <typename I>
void AbstractObjectWriteRequest<I>::write_object() {
  ...
  librados::ObjectWriteOperation write; // 创建一个librados::ObjectWriteOperation
  ...
  add_write_ops(&write); // 为新创建的librados::ObjectWriteOperation填充相应的ObjectOperation
  ...
  int r = image_ctx->data_ctx.aio_operate( // 调用data_ctx.aio_operate()处理write操作
    data_object_name(this->m_ictx, this->m_object_no), rados_completion, // 计算对象名
    &write, m_snap_seq, m_snaps,
    (this->m_trace.valid() ? this->m_trace.get_info() : nullptr));
  ...
  rados_completion->release();
}
```

ObjectWriteRequest.add_write_ops()方法

```C++
template <typename I>
void ObjectWriteRequest<I>::add_write_ops(librados::ObjectWriteOperation *wr) {
  if (this->m_full_object) { // 如果需要写入整个对象
    wr->write_full(m_write_data);
  } else { // 如果只写入对象的部分范围
    wr->write(this->m_object_off, m_write_data); // 带入offset参数，调用ObjectWriteOperation的write方法
  }
  wr->set_op_flags2(m_op_flags); // 为ObjectWriteOperation设置op的flag
}
```

涉及到的主要数据结构及他们之间的关系:

![](.\src\ObjectOperationClass.drawio.png)

整个ObjectWriteOperation.write()方法内部的工作流程及调用栈：

```C++
// ObjectWriteOperation.write方法()
void librados::ObjectWriteOperation::write(uint64_t off, const bufferlist& bl)
{
  ceph_assert(impl);
  ::ObjectOperation *o = &impl->o; 
  bufferlist c = bl;
  o->write(off, c);
}


// src/osdc/Objecter.h
void write(uint64_t off, ceph::buffer::list& bl) {
  write(off, bl, 0, 0);
}

// src/osdc/Objecter.h
void write(uint64_t off, ceph::buffer::list& bl,
           uint64_t truncate_size,
           uint32_t truncate_seq) {
  add_data(CEPH_OSD_OP_WRITE, off, bl.length(), bl);
  OSDOp& o = *ops.rbegin();
  o.op.extent.truncate_size = truncate_size;
  o.op.extent.truncate_seq = truncate_seq;
}

// src/osdc/Objecter.h
void add_data(int op, uint64_t off, uint64_t len, ceph::buffer::list& bl) {
  OSDOp& osd_op = add_op(op); // 根据op类型(此例中为CEPH_OSD_OP_WRITE)新增一个OSDOp，并返回其引用
  osd_op.op.extent.offset = off; // 设置op.extent的offset
  osd_op.op.extent.length = len; // 设置op.extent的length
  osd_op.indata.claim_append(bl); // 填充输入数据
}

// src/osd/osd_types.h
OSDOp& add_op(int op) {
  int s = ops.size(); // 获取当前OSDOp向量的长度
  ops.resize(s+1); // 新增一个OSDOp
  ops[s].op.op = op; // 填充op类型
  out_bl.resize(s+1);
  out_bl[s] = NULL;
  out_handler.resize(s+1);
  out_handler[s] = NULL;
  out_rval.resize(s+1);
  out_rval[s] = NULL;
  return ops[s]; // 返回新增的OSDOp
}
```

ObjectWriteRequest.data_object_name()方法

根据对象的编号，计算对象名

```c++
template <typename I>
std::string data_object_name(I* image_ctx, uint64_t object_no) {
  char buf[RBD_MAX_OBJ_NAME_SIZE]; // #define RBD_MAX_OBJ_NAME_SIZE	96
  size_t length = snprintf(buf, RBD_MAX_OBJ_NAME_SIZE,
    image_ctx->format_string, // 例如: rbd_data.1104d8944d6a.%016llx, 前半部分为object_prefix, open image时从rbd_header读取，后半部分为字符串的填充格式定义
    object_no); // 对象在镜像地址空间内的编号
  ...
}
```

主要工作内容:

* 先对无op且对象不存在的请求进行优化，即直接结束处理
* 接着如果是克隆的镜像，当被访问对象还不存在时，调用copyup()拷贝对象数据到本地镜像中
* 再创建一个librados::ObjectWriteOperation对象，并填充相应的ObjectOperation。ObjectOperation是对底层数据结构ceph_osd_op的层层封装。
* 因为是数据I/O，则调用镜像上下文(image_ctx)的数据存储池上下文(data_ctx)的aio_operate()方法继续处理。



## 写请求在librados层的处理

经过上一节的处理，块层的业务逻辑至此结束，交由librados库中继续处理。

### 主要工作内容

* librados层这里的处理比较简单，主要是根据ObjectOperation中的一组OSDOp构造osdc层的Op对象。
* 然后执行Objecter.op_submit()方法处理该Op对象

### 调用流程

librados::IoCtx.aio_operate()方法

```c++
int librados::IoCtx::aio_operate(const std::string& oid, AioCompletion *c,
         librados::ObjectWriteOperation *o,
         snap_t snap_seq, std::vector<snap_t>& snaps,
         const blkin_trace_info *trace_info)
{
  ... // SnapContext相关处理
  return io_ctx_impl->aio_operate(obj, &o->impl->o, c->pc,
          snapc, 0, trace_info);
}
```

librados::IoCtxImpl.aio_operate()方法

```c++
int librados::IoCtxImpl::aio_operate(const object_t& oid,
				     ::ObjectOperation *o, AioCompletionImpl *c,
				     const SnapContext& snap_context, int flags,
                                     const blkin_trace_info *trace_info)
{
  ...
  /* can't write to a snapshot */
  if (snap_seq != CEPH_NOSNAP)
    return -EROFS;
  ...
  c->io = this;
  queue_aio_write(c); // 把AioCompletionImpl中的aio_write_list_item加入到this->aio_write_list中
  ...
  Objecter::Op *op = objecter->prepare_mutate_op(
    oid, oloc, *o, snap_context, ut, flags,
    oncomplete, &c->objver, osd_reqid_t(), &trace);
  objecter->op_submit(op, &c->tid);
  ...
  return 0;
}
```

Objecter.prepare_mutate_op()方法

```c++
Op *prepare_mutate_op(
  const object_t& oid, const object_locator_t& oloc,
  ObjectOperation& op, const SnapContext& snapc,
  ceph::real_time mtime, int flags,
  Context *oncommit, version_t *objver = NULL,
  osd_reqid_t reqid = osd_reqid_t(),
  ZTracer::Trace *parent_trace = nullptr) {
  // 根据对象名, ObjectOperation.ops等信息构建Objecter::Op*类型对象op
  Op *o = new Op(oid, oloc, op.ops, flags | global_op_flags |
	   CEPH_OSD_FLAG_WRITE, oncommit, objver, nullptr, parent_trace);
  // 填充其他属性
  o->priority = op.priority;
  o->mtime = mtime;
  o->snapc = snapc;
  o->out_rval.swap(op.out_rval);
  o->out_bl.swap(op.out_bl);
  o->out_handler.swap(op.out_handler);
  o->reqid = reqid;
  return o;
}
```



## 写请求在osdc层的处理

osdc层负责对请求的封装，以及通过网络模块发送请求的工作。

### 主要工作内容

* Objecter中先做了流控、超时方面的处理
* 然后根据存储池配置、crushmap计算出object所在的主OSD
* 根据OSD的ID获取一个已有的或者建立一个新的会话，其中最主要的是利用messenger.connect_to_osd()建立一个套接字连接
* 再把Objector::Op请求转换成下层的消息层所使用的MOSDOp
* 最后使用会话中的连接把该消息发送出去，至此完成写I/O的主要调用流程

### Objecter类定义

objecter是osdc中的一个核心类

```c++
class Objecter : public md_config_obs_t, public Dispatcher {
...
public:
  Messenger *messenger; // 消息模块
  MonClient *monc; // Monitor客户端
  ...
private:
  std::unique_ptr<OSDMap> osdmap; // 指向osdmap
...
public:
  /*** track pending operations ***/
  // read

  struct OSDSession; // OSD会话

  struct op_target_t {
    int flags = 0;

    epoch_t epoch = 0;  ///< latest epoch we calculated the mapping
    ...
    ///< explcit pg target, if any
    pg_t base_pgid;
    ...
    int osd = -1;      ///< the final target osd, or -1
	...
  };

  struct Op : public RefCountedObject {
    OSDSession *session; // 关联的OSD会话
    ...
    op_target_t target; // op的目标结构体，主要定义了osd号、pgid、epoch号
    ...
    std::vector<OSDOp> ops; // 一组OSDOp
    ...
    int priority; // op的优先级
    ...
  }
  ...
};
```

### 调用流程

Objecter.op_submit()方法

```c++
void Objecter::op_submit(Op *op, ceph_tid_t *ptid, int *ctx_budget)
{
  ...
  _op_submit_with_budget(op, rl, ptid, ctx_budget);
}
```

Objecter._op_submit_with_budget()方法

主要包含对op的流量控制和超时处理

```c++
void Objecter::_op_submit_with_budget(Op *op, shunique_lock& sul,
				      ceph_tid_t *ptid,
				      int *ctx_budget)
{
  ...
  // throttle.  before we look at any state, because
  // _take_op_budget() may drop our lock while it blocks.
  if (!op->ctx_budgeted || (ctx_budget && (*ctx_budget == -1))) {
    int op_budget = _take_op_budget(op, sul);
    // take and pass out the budget for the first OP
    // in the context session
    if (ctx_budget && (*ctx_budget == -1)) {
      *ctx_budget = op_budget;
    }
  }

  if (osd_timeout > timespan(0)) {
    if (op->tid == 0)
      op->tid = ++last_tid;
    auto tid = op->tid;
    op->ontimeout = timer.add_event(osd_timeout, // 设置op的超时时间, 及相应的回调函数
				    [this, tid]() {
				      op_cancel(tid, -ETIMEDOUT); });
  }

  _op_submit(op, sul, ptid);
}
```

Objecter._op_submit()方法

op_submit()方法十分重要，最主要的是使用_calc_target()方法，对object经过一系列计算，得出其所在的pg及osd，从而决定此op发往哪一个osd，具体细节这里先不展开。

```c++
void Objecter::_op_submit(Op *op, shunique_lock& sul, ceph_tid_t *ptid)
{
  ...
  OSDSession *s = NULL;
  bool check_for_latest_map = _calc_target(&op->target, nullptr) // 计算该object所在的主OSD, 并填充到op->target.osd
    == RECALC_OP_TARGET_POOL_DNE;

  // Try to get a session, including a retry if we need to take write lock
  int r = _get_session(op->target.osd, &s, sul);
  ...
  _send_op_account(op); // 根据op的内容添加审计日志
  ...
  _session_op_assign(s, op); // 把该op与OSDSession进行绑定

  if (need_send) {
    _send_op(op); // 将op发送出去
  }
  ...
}
```

Objecter::_get_session()方法

```c++
int Objecter::_get_session(int osd, OSDSession **session, shunique_lock& sul)
{
  ...
  map<int,OSDSession*>::iterator p = osd_sessions.find(osd); // 首先在已经创建的OSDSession表中根据osd_id进行查找
  if (p != osd_sessions.end()) {
    OSDSession *s = p->second;
    s->get();
    *session = s;
    ldout(cct, 20) << __func__ << " s=" << s << " osd=" << osd << " "
		   << s->get_nref() << dendl;
    return 0;
  }
  ...
  OSDSession *s = new OSDSession(cct, osd); // 新建一个OSDSession
  osd_sessions[osd] = s; // 把新的OSDSession加入OSDSession表中
  s->con = messenger->connect_to_osd(osdmap->get_addrs(osd)); // 根据osd的地址, 调用messenger->connect_to_osd获取连接
  ...
  s->get();
  *session = s; // 返回新OSDSession的地址
  ldout(cct, 20) << __func__ << " s=" << s << " osd=" << osd << " "
		 << s->get_nref() << dendl;
  return 0;
}
```

Objecter._send_op()方法

```c++
void Objecter::_send_op(Op *op)
{
  ...
  MOSDOp *m = _prepare_osd_op(op); // 根据Op创建一个MOSDOp, 并进行填充

  if (op->target.actual_pgid != m->get_spg()) {
    ...
    m->set_spg(op->target.actual_pgid); // 填充pgid
    m->clear_payload();  // reencode
  }
  ...
  op->session->con->send_message(m); // 根据已创建的连接发送该MOSDOp消息
}
```

至此，整个librbd客户端的写I/O工作流程就已经完成，等到目标OSD处理完写I/O业务并返回时，客户端又会层层回调，最终ImageRequestWQ.write()中的
cond.wait()返回，今儿完成了一次Image.write()操作。



## 参考

* [source code](https://github.com/ceph/ceph/tree/main/src/librbd)

* [Ceph源码解析：读写流程]( https://www.cnblogs.com/chenxianpao/p/5572859.html)
