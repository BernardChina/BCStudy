####GCD Queue源码分析
`dispatch_queue_s` : `GCD`中的`queue`。

`dispatch_continuation_s` : 我们向`queue`提交的任务，无论`block`还是`function`形式，最终都会被封装为`dispatch_continuation_s`。

使用GCD，需要把任务提交到队列。而我们可以用如下三种方法之一获取要使用的队列：

```objectivec
dispatch_queue_t
dispatch_queue_create(const char *_Nullable label,
        dispatch_queue_attr_t _Nullable attr)

dispatch_queue_t dispatch_get_main_queue(void)

dispatch_queue_t
dispatch_get_global_queue(long identifier, unsigned long flags)

---------------------

```

第一种方法是创建一个`queue`，后两种方法是获取系统自定义的`queue`。

但其实背后的实现是，`无论是自己创建还是获取系统定义的queue，只会在GCD启动时创建的root queue数组中，取得一个queue而已`。

如何创建root queue?

```
//我们在源码中大致看到这样的代码
void
dispatch_main(void)
{
	_dispatch_root_queues_init();
}
```

`root queue`一共有`12`个，分别有不同的优先级和序列号。我们首先来看一下，这个12个`root queue`如何定义的:

```cpp
DISPATCH_CACHELINE_ALIGN
struct dispatch_queue_s _dispatch_root_queues[] = {
#define _DISPATCH_ROOT_QUEUE_IDX(n, flags) \
	((flags & DISPATCH_PRIORITY_FLAG_OVERCOMMIT) ? \
		DISPATCH_ROOT_QUEUE_IDX_##n##_QOS_OVERCOMMIT : \
		DISPATCH_ROOT_QUEUE_IDX_##n##_QOS)
#define _DISPATCH_ROOT_QUEUE_ENTRY(n, flags, ...) \
	[_DISPATCH_ROOT_QUEUE_IDX(n, flags)] = { \
		DISPATCH_GLOBAL_OBJECT_HEADER(queue_root), \
		.dq_state = DISPATCH_ROOT_QUEUE_STATE_INIT_VALUE, \
		.do_ctxt = &_dispatch_root_queue_contexts[ \
				_DISPATCH_ROOT_QUEUE_IDX(n, flags)], \
		.dq_atomic_flags = DQF_WIDTH(DISPATCH_QUEUE_WIDTH_POOL), \
		.dq_priority = _dispatch_priority_make(DISPATCH_QOS_##n, 0) | flags | \
				DISPATCH_PRIORITY_FLAG_ROOTQUEUE | \
				((flags & DISPATCH_PRIORITY_FLAG_DEFAULTQUEUE) ? 0 : \
				DISPATCH_QOS_##n << DISPATCH_PRIORITY_OVERRIDE_SHIFT), \
		__VA_ARGS__ \
	}
	_DISPATCH_ROOT_QUEUE_ENTRY(MAINTENANCE, 0,
		.dq_label = "com.apple.root.maintenance-qos",
		.dq_serialnum = 4,
	),
	_DISPATCH_ROOT_QUEUE_ENTRY(MAINTENANCE, DISPATCH_PRIORITY_FLAG_OVERCOMMIT,
		.dq_label = "com.apple.root.maintenance-qos.overcommit",
		.dq_serialnum = 5,
	),
	_DISPATCH_ROOT_QUEUE_ENTRY(BACKGROUND, 0,
		.dq_label = "com.apple.root.background-qos",
		.dq_serialnum = 6,
	),
	_DISPATCH_ROOT_QUEUE_ENTRY(BACKGROUND, DISPATCH_PRIORITY_FLAG_OVERCOMMIT,
		.dq_label = "com.apple.root.background-qos.overcommit",
		.dq_serialnum = 7,
	),
	_DISPATCH_ROOT_QUEUE_ENTRY(UTILITY, 0,
		.dq_label = "com.apple.root.utility-qos",
		.dq_serialnum = 8,
	),
	_DISPATCH_ROOT_QUEUE_ENTRY(UTILITY, DISPATCH_PRIORITY_FLAG_OVERCOMMIT,
		.dq_label = "com.apple.root.utility-qos.overcommit",
		.dq_serialnum = 9,
	),
	_DISPATCH_ROOT_QUEUE_ENTRY(DEFAULT, DISPATCH_PRIORITY_FLAG_DEFAULTQUEUE,
		.dq_label = "com.apple.root.default-qos",
		.dq_serialnum = 10,
	),
	_DISPATCH_ROOT_QUEUE_ENTRY(DEFAULT,
			DISPATCH_PRIORITY_FLAG_DEFAULTQUEUE | DISPATCH_PRIORITY_FLAG_OVERCOMMIT,
		.dq_label = "com.apple.root.default-qos.overcommit",
		.dq_serialnum = 11,
	),
	_DISPATCH_ROOT_QUEUE_ENTRY(USER_INITIATED, 0,
		.dq_label = "com.apple.root.user-initiated-qos",
		.dq_serialnum = 12,
	),
	_DISPATCH_ROOT_QUEUE_ENTRY(USER_INITIATED, DISPATCH_PRIORITY_FLAG_OVERCOMMIT,
		.dq_label = "com.apple.root.user-initiated-qos.overcommit",
		.dq_serialnum = 13,
	),
	_DISPATCH_ROOT_QUEUE_ENTRY(USER_INTERACTIVE, 0,
		.dq_label = "com.apple.root.user-interactive-qos",
		.dq_serialnum = 14,
	),
	_DISPATCH_ROOT_QUEUE_ENTRY(USER_INTERACTIVE, DISPATCH_PRIORITY_FLAG_OVERCOMMIT,
		.dq_label = "com.apple.root.user-interactive-qos.overcommit",
		.dq_serialnum = 15,
	),
};
```

让我们一起看看它们的实现，首先，是`dispatch_queue_create`。

```
dispatch_queue_t
dispatch_queue_create(const char *label, dispatch_queue_attr_t attr)
{
    return _dispatch_queue_create_with_target(label, attr,
            DISPATCH_TARGET_QUEUE_DEFAULT, true);
}

---------------------
```
当用户要创建一个`queue`的时候，需要指定`target queue`，即创建的`queue`最终是在哪个`queue`上执行的，这称之为`target queue`。对于用户创建的`queue`来说，这个`target queue`会取`root queue`之一。

这里的`DISPATCH_TARGET_QUEUE_DEFAULT`是一个宏定义：
```
#define DISPATCH_TARGET_QUEUE_DEFAULT NULL
```

```
static dispatch_queue_t
_dispatch_queue_create_with_target(const char *label, dispatch_queue_attr_t dqa,
        dispatch_queue_t tq, bool legacy)
{
    if (!slowpath(dqa)) { // 如果是串行队列
        dqa = _dispatch_get_default_queue_attr(); // 串行队列的attr 取默认attr
    } else if (dqa->do_vtable != DISPATCH_VTABLE(queue_attr)) { // 并行队列的 attr->do_vtable 应该等于 DISPATCH_VTABLE(queue_attr)
        DISPATCH_CLIENT_CRASH(dqa->do_vtable, "Invalid queue attribute");
    }

    //
    // Step 1: Normalize arguments (qos, overcommit, tq)
    //

    // dispatch_qos_t qos是优先级？
    dispatch_qos_t qos = _dispatch_priority_qos(dqa->dqa_qos_and_relpri);
    // 是否overcommit（即queue创建的线程数是否允许超过实际的CPU个数）
    _dispatch_queue_attr_overcommit_t overcommit = dqa->dqa_overcommit;
    if (overcommit != _dispatch_queue_attr_overcommit_unspecified && tq) { //
        if (tq->do_targetq) { // overcommit 的queue 必须是全局的
            DISPATCH_CLIENT_CRASH(tq, "Cannot specify both overcommit and "
                    "a non-global target queue");
        }
    }

    // 下面这些代码，因为用户创建的queue的tq一定为NULL，因此，只要关注tq == NULL的分支即可，我们删除了其余分支
    if (!tq) { // 自己创建的queue，tq都是null
        tq = _dispatch_get_root_queue( // 在root queue里面去取一个合适的queue当做target queue
                qos == DISPATCH_QOS_UNSPECIFIED ? DISPATCH_QOS_DEFAULT : qos, // 无论是用户创建的串行还是并行队列，其qos都没有指定，因此，qos这里都取DISPATCH_QOS_DEFAULT
                overcommit == _dispatch_queue_attr_overcommit_enabled);
        if (slowpath(!tq)) { // 如果根据create queue是传入的属性无法获取到对应的tq，crash
            DISPATCH_CLIENT_CRASH(qos, "Invalid queue attribute");
        }
    }

    //
    // Step 2: Initialize the queue
    //
    const void *vtable;
    dispatch_queue_flags_t dqf = 0;
    // 根据不同的queue类型，设置vtable。vtable实现了SERIAL queue 和 CONCURRENT queue的行为差异。
    if (dqa->dqa_concurrent) { // 并发
        vtable = DISPATCH_VTABLE(queue_concurrent);
    } else {  // 串行
        vtable = DISPATCH_VTABLE(queue_serial);
    }

    // 创建一个与tq对应的dq，用来给用户返回
    dispatch_queue_t dq = _dispatch_object_alloc(vtable,
            sizeof(struct dispatch_queue_s) - DISPATCH_QUEUE_CACHELINE_PAD);
    // 初始化dq，可以看到dqa->dqa_concurrent，对于并发队列，其queue width是DISPATCH_QUEUE_WIDTH_MAX，而串行队列其width是1
    _dispatch_queue_init(dq, dqf, dqa->dqa_concurrent ? 
            DISPATCH_QUEUE_WIDTH_MAX : 1, DISPATCH_QUEUE_ROLE_INNER |
            (dqa->dqa_inactive ? DISPATCH_QUEUE_INACTIVE : 0));

    dq->dq_label = label; // 设置dq的名字
#if HAVE_PTHREAD_WORKQUEUE_QOS
    dq->dq_priority = dqa->dqa_qos_and_relpri;
    if (overcommit == _dispatch_queue_attr_overcommit_enabled) {
        dq->dq_priority |= DISPATCH_PRIORITY_FLAG_OVERCOMMIT;
    }
#endif
    _dispatch_retain(tq);
    if (qos == QOS_CLASS_UNSPECIFIED) { // 如果没有指定queue的优先级，则默认继承target queue的优先级
        // legacy way of inherithing the QoS from the target
        _dispatch_queue_priority_inherit_from_target(dq, tq);
    }
    if (!dqa->dqa_inactive) {
        _dispatch_queue_inherit_wlh_from_target(dq, tq);
    }
    dq->do_targetq = tq; // 这一步，很关键！！！ 将root queue设置为dq的target queue，root queue和新创建的queue联合在了一起
    _dispatch_object_debug(dq, "%s", __func__);
    // 将新创建的dq，添加到GCD内部管理的叫做_dispatch_introspection的queue列表中。这是GCD内部维护的一个queue列表，具体作用不太清楚。
    return _dispatch_introspection_queue_create(dq);
}

---------------------
```
去除多余的判断，其实逻辑也很简单，当我们调用`dispatch_queue_create`，`GCD`内部会调用`_dispatch_queue_create_with_target`， 它首先会根据我们创建的queue的属性：`DISPATCH_QUEUE_SERIAL`或`DISPATCH_QUEUE_CONCURRENT`，到`root queue`数组中取出一个对应的`queue`作为`target queue`。然后，会新建一个`dispatch_queue`_t对象，并设置其`target queue`，返回给用户。同时，在`GCD`内部，新建的`queue`还会被加入`introspection queue`列表中。

这里的关键是根据queue属性获取对应的target root queue：

```
tq = _dispatch_get_root_queue( // 在root queue里面去取一个合适的queue当做target queue
                qos == DISPATCH_QOS_UNSPECIFIED ? DISPATCH_QOS_DEFAULT : qos, // 串行 用DISPATCH_QOS_DEFAULT， 并行 用自己的qos

```
`_dispatch_get_root_queue`的实现如下：

```objectivec
static inline dispatch_queue_t
_dispatch_get_root_queue(dispatch_qos_t qos, bool overcommit)
{
    return &_dispatch_root_queues[2 * (qos - 1) + overcommit]; // 根据qos和 是否overcommit，取得root queues数组中对应的queue(一共有12个root queue)
}
 
```

`qos==DISPATCH_QOS_DEFAULT`，值是4。根据公式，2*(4-1) + 1 = 7, 所有创建的串行队列应该取root queue数组的第8个queue，而由于并行队列的`overcommit`是`false（0）`，并行队列回去root queue的第`7`个元素

```objectivec
// 用户创建的并行队列，默认会使用这个target queue
    _DISPATCH_ROOT_QUEUE_ENTRY(DEFAULT, DISPATCH_PRIORITY_FLAG_DEFAULTQUEUE,
        .dq_label = "com.apple.root.default-qos",
        .dq_serialnum = 10,
    ),
    // 用户创建的串行队列，会使这个target queue, main queue 也会取这个值，但是会将 .dq_label = "com.apple.main-thread",
                                                                        // .dq_atomic_flags = DQF_THREAD_BOUND | DQF_CANNOT_TRYSYNC | DQF_WIDTH(1),
                                                                        // .dq_serialnum = 1,
    _DISPATCH_ROOT_QUEUE_ENTRY(DEFAULT,
            DISPATCH_PRIORITY_FLAG_DEFAULTQUEUE | DISPATCH_PRIORITY_FLAG_OVERCOMMIT,
        .dq_label = "com.apple.root.default-qos.overcommit",
        .dq_serialnum = 11,
    ),

```
OK，现在我们已经知道用户自创建的queue默认都会附加到root queue上。那么对于dispatch_get_main_queue, dispatch_get_global_queue是否也有类似的逻辑呢？我们先来看dispatch_get_global_queue。

#####dispatch_get_global_queue
我们通常会用如下代码，来获取一个全局的并发队列，并指定其优先级。
```
dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)
```
实现如下：

```
dispatch_queue_t
dispatch_get_global_queue(long priority, unsigned long flags)
{
    if (flags & ~(unsigned long)DISPATCH_QUEUE_OVERCOMMIT) {
        return DISPATCH_BAD_INPUT;
    }
    dispatch_qos_t qos = _dispatch_qos_from_queue_priority(priority); // 将用户使用的优先级转换为root queue的优先级
    if (qos == DISPATCH_QOS_UNSPECIFIED) {
        return DISPATCH_BAD_INPUT;
    }
    return _dispatch_get_root_queue(qos, flags & DISPATCH_QUEUE_OVERCOMMIT); // 由于flags是保留值，均取0，因此global queue都是no overcommit的
}

```
逻辑很简单，首先将外部传入的queue优先级转换为GCD内部的优先级`dispatch_qos_t qos`。 然后，在调用`_dispatch_get_root_queue` 获取`root queue`中对应的`queue`。

我们来看一下`_dispatch_qos_from_queue_priority`的实现：

```
static inline dispatch_qos_t
_dispatch_qos_from_queue_priority(long priority)
{
    switch (priority) {
    case DISPATCH_QUEUE_PRIORITY_BACKGROUND:      return DISPATCH_QOS_BACKGROUND;
    case DISPATCH_QUEUE_PRIORITY_NON_INTERACTIVE: return DISPATCH_QOS_UTILITY;
    case DISPATCH_QUEUE_PRIORITY_LOW:             return DISPATCH_QOS_UTILITY;
    case DISPATCH_QUEUE_PRIORITY_DEFAULT:         return DISPATCH_QOS_DEFAULT;
    case DISPATCH_QUEUE_PRIORITY_HIGH:            return DISPATCH_QOS_USER_INITIATED;
    default: return _dispatch_qos_from_qos_class((qos_class_t)priority);
    }
}

```

#####dispatch_get_main_queue

```
/*!
 * @function dispatch_get_main_queue
 *
 * @abstract
 * Returns the default queue that is bound to the main thread.
 *
 * @discussion
 * In order to invoke blocks submitted to the main queue, the application must
 * call dispatch_main(), NSApplicationMain(), or use a CFRunLoop on the main
 * thread.
 *
 * @result
 * Returns the main queue. This queue is created automatically on behalf of
 * the main thread before main() is called.
 */
dispatch_queue_t
dispatch_get_main_queue(void)
{
    return DISPATCH_GLOBAL_OBJECT(dispatch_queue_t, _dispatch_main_q);
}

```
根据注释，可以看到，`main queue`是由系统在`main()`方法调用前创建的。专门绑定到`main thread`上。而且，为了触发提交到`main queue`上的`block`，和其他`queue`不一样，`main queue`上的任务是依赖于`main runloop`触发的。

#####_dispatch_main_q
```
struct dispatch_queue_s _dispatch_main_q = {
    DISPATCH_GLOBAL_OBJECT_HEADER(queue_main),
#if !DISPATCH_USE_RESOLVERS
    .do_targetq = &_dispatch_root_queues[
            DISPATCH_ROOT_QUEUE_IDX_DEFAULT_QOS_OVERCOMMIT],  // 同样也是取root queue中的queue作为target queue
#endif
    .dq_state = DISPATCH_QUEUE_STATE_INIT_VALUE(1) |
            DISPATCH_QUEUE_ROLE_BASE_ANON,
    .dq_label = "com.apple.main-thread",
    .dq_atomic_flags = DQF_THREAD_BOUND | DQF_CANNOT_TRYSYNC | DQF_WIDTH(1),
    .dq_serialnum = 1,
};

```

####总结
从上面源码可以看出，GCD用到的queue，无论是自己创建的，或是获取系统的main queue还是global queue，其最终都是落脚于GCD root queue中。  我们可以在代码中的任意位置创建queue，但最终GCD管理的，也不过这12个root queue，这种思路有点类似于命令模式，即分散创建，集中管理。

#####备注
在我们创建queue的时候，其实是为了后面，调用disptch_asyc dispatch_syc. 不同的queue执行的方法也不同，下面我来看一下定义：
```
#pragma mark dispatch_vtables

DISPATCH_VTABLE_INSTANCE(semaphore,
	.do_type = DISPATCH_SEMAPHORE_TYPE,
	.do_kind = "semaphore",
	.do_dispose = _dispatch_semaphore_dispose,
	.do_debug = _dispatch_semaphore_debug,
);

DISPATCH_VTABLE_INSTANCE(group,
	.do_type = DISPATCH_GROUP_TYPE,
	.do_kind = "group",
	.do_dispose = _dispatch_group_dispose,
	.do_debug = _dispatch_group_debug,
);

DISPATCH_VTABLE_INSTANCE(queue,
	.do_type = DISPATCH_QUEUE_LEGACY_TYPE,
	.do_kind = "queue",
	.do_dispose = _dispatch_queue_dispose,
	.do_suspend = _dispatch_queue_suspend,
	.do_resume = _dispatch_queue_resume,
	.do_push = _dispatch_queue_push,
	.do_invoke = _dispatch_queue_invoke,
	.do_wakeup = _dispatch_queue_wakeup,
	.do_debug = dispatch_queue_debug,
	.do_set_targetq = _dispatch_queue_set_target_queue,
);

DISPATCH_VTABLE_SUBCLASS_INSTANCE(queue_serial, queue,
	.do_type = DISPATCH_QUEUE_SERIAL_TYPE,
	.do_kind = "serial-queue",
	.do_dispose = _dispatch_queue_dispose,
	.do_suspend = _dispatch_queue_suspend,
	.do_resume = _dispatch_queue_resume,
	.do_finalize_activation = _dispatch_queue_finalize_activation,
	.do_push = _dispatch_queue_push,
	.do_invoke = _dispatch_queue_invoke,
	.do_wakeup = _dispatch_queue_wakeup,
	.do_debug = dispatch_queue_debug,
	.do_set_targetq = _dispatch_queue_set_target_queue,
);

DISPATCH_VTABLE_SUBCLASS_INSTANCE(queue_concurrent, queue,
	.do_type = DISPATCH_QUEUE_CONCURRENT_TYPE,
	.do_kind = "concurrent-queue",
	.do_dispose = _dispatch_queue_dispose,
	.do_suspend = _dispatch_queue_suspend,
	.do_resume = _dispatch_queue_resume,
	.do_finalize_activation = _dispatch_queue_finalize_activation,
	.do_push = _dispatch_queue_push,
	.do_invoke = _dispatch_queue_invoke,
	.do_wakeup = _dispatch_queue_wakeup,
	.do_debug = dispatch_queue_debug,
	.do_set_targetq = _dispatch_queue_set_target_queue,
);


DISPATCH_VTABLE_SUBCLASS_INSTANCE(queue_root, queue,
	.do_type = DISPATCH_QUEUE_GLOBAL_ROOT_TYPE,
	.do_kind = "global-queue",
	.do_dispose = _dispatch_pthread_root_queue_dispose,
	.do_push = _dispatch_root_queue_push,
	.do_invoke = NULL,
	.do_wakeup = _dispatch_root_queue_wakeup,
	.do_debug = dispatch_queue_debug,
);


DISPATCH_VTABLE_SUBCLASS_INSTANCE(queue_main, queue,
	.do_type = DISPATCH_QUEUE_SERIAL_TYPE,
	.do_kind = "main-queue",
	.do_dispose = _dispatch_queue_dispose,
	.do_push = _dispatch_queue_push,
	.do_invoke = _dispatch_queue_invoke,
	.do_wakeup = _dispatch_main_queue_wakeup,
	.do_debug = dispatch_queue_debug,
);

DISPATCH_VTABLE_SUBCLASS_INSTANCE(queue_runloop, queue,
	.do_type = DISPATCH_QUEUE_RUNLOOP_TYPE,
	.do_kind = "runloop-queue",
	.do_dispose = _dispatch_runloop_queue_dispose,
	.do_push = _dispatch_queue_push,
	.do_invoke = _dispatch_queue_invoke,
	.do_wakeup = _dispatch_runloop_queue_wakeup,
	.do_debug = dispatch_queue_debug,
);

DISPATCH_VTABLE_SUBCLASS_INSTANCE(queue_mgr, queue,
	.do_type = DISPATCH_QUEUE_MGR_TYPE,
	.do_kind = "mgr-queue",
	.do_push = _dispatch_mgr_queue_push,
	.do_invoke = _dispatch_mgr_thread,
	.do_wakeup = _dispatch_mgr_queue_wakeup,
	.do_debug = dispatch_queue_debug,
);

DISPATCH_VTABLE_INSTANCE(queue_specific_queue,
	.do_type = DISPATCH_QUEUE_SPECIFIC_TYPE,
	.do_kind = "queue-context",
	.do_dispose = _dispatch_queue_specific_queue_dispose,
	.do_push = (void *)_dispatch_queue_push,
	.do_invoke = (void *)_dispatch_queue_invoke,
	.do_wakeup = (void *)_dispatch_queue_wakeup,
	.do_debug = (void *)dispatch_queue_debug,
);

DISPATCH_VTABLE_INSTANCE(queue_attr,
	.do_type = DISPATCH_QUEUE_ATTR_TYPE,
	.do_kind = "queue-attr",
);

DISPATCH_VTABLE_INSTANCE(source,
	.do_type = DISPATCH_SOURCE_KEVENT_TYPE,
	.do_kind = "kevent-source",
	.do_dispose = _dispatch_source_dispose,
	.do_suspend = (void *)_dispatch_queue_suspend,
	.do_resume = (void *)_dispatch_queue_resume,
	.do_finalize_activation = _dispatch_source_finalize_activation,
	.do_push = (void *)_dispatch_queue_push,
	.do_invoke = _dispatch_source_invoke,
	.do_wakeup = _dispatch_source_wakeup,
	.do_debug = _dispatch_source_debug,
	.do_set_targetq = (void *)_dispatch_queue_set_target_queue,
);
```
在我们讲到其他GCD源码分析-异步同步执行的时候，会用到。

>参考文章https://blog.csdn.net/u013378438/article/details/81031938
