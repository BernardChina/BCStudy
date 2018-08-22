#####NSProxy的使用

总结：
总体来讲，`NSProxy`主要服务于消息转发使用，通过它，我们可以做到组件的开发，彼此之间的节藕。切面编程(AOP)，也是通过此方法。


我们大家都知`NSObject`是什么，基本上所有的类都是继承于`NSObject`的。我们首先来看一下`NSObject`的定义
```
@interface NSObject <NSObject> {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wobjc-interface-ivars"
    Class isa  OBJC_ISA_AVAILABILITY;
#pragma clang diagnostic pop
}

+ (void)load;

+ (void)initialize;
- (instancetype)init
#if NS_ENFORCE_NSOBJECT_DESIGNATED_INITIALIZER
    NS_DESIGNATED_INITIALIZER
#endif
    ;

+ (instancetype)new OBJC_SWIFT_UNAVAILABLE("use object initializers instead");
+ (instancetype)allocWithZone:(struct _NSZone *)zone OBJC_SWIFT_UNAVAILABLE("use object initializers instead");
+ (instancetype)alloc OBJC_SWIFT_UNAVAILABLE("use object initializers instead");
- (void)dealloc OBJC_SWIFT_UNAVAILABLE("use 'deinit' to define a de-initializer");
....
....
....
//后面有很多代码
```
我们从代码中看到，`NSObject`也是实现了`<NSObject>`协议的类，只不过自己同时定义了`init`，`alloc`，`copy`等方法。
那么，我们下面来看一下`NSProxy`的具体定义，它到底是什么
```
@interface NSProxy <NSObject> {
    Class	isa;
}

+ (id)alloc;
+ (id)allocWithZone:(nullable NSZone *)zone NS_AUTOMATED_REFCOUNT_UNAVAILABLE;
+ (Class)class;

- (void)forwardInvocation:(NSInvocation *)invocation;
- (nullable NSMethodSignature *)methodSignatureForSelector:(SEL)sel NS_SWIFT_UNAVAILABLE("NSInvocation and related APIs not available");
- (void)dealloc;
- (void)finalize;
@property (readonly, copy) NSString *description;
@property (readonly, copy) NSString *debugDescription;
+ (BOOL)respondsToSelector:(SEL)aSelector;

- (BOOL)allowsWeakReference NS_UNAVAILABLE;
- (BOOL)retainWeakReference NS_UNAVAILABLE;

// - (id)forwardingTargetForSelector:(SEL)aSelector;

@end
```
我们发现其实，`NSProxy`和`NSObject`是同一种级别的类，都是实现了`<NSObject>`协议。只不过不同的是，`NSProxy`没有`init`方法，通过`NSProxy`的名字和具体 定义，我们也能看出来，其实`NSProxy`主重要作用，是做消息转发，因为它没有定义其他的方法。

#####NSProxy的使用技巧
通过上面的定义，我们基本知道，`NSProxy`主要作用是消息转发，下面我们来看一下，如何做到消息转发的。
```
@interface MyProxy : NSProxy

@property (assign, nonatomic) __unsafe_unretained id representedObject; //被代理的类，可以理解为target

+ (instancetype)proxyForObject:(id)object;

@end
```
首先，我们定义了自己的代理类`myProxy`，它继承于`NSProxy`，其实和我们平常用`NSObject`没有什么区别(当然还是有区别的，下面我们介绍)
```
@implementation MyProxy

+ (instancetype)proxyForObject:(id)object
{
//通过nsproxy的定义，我们知道，它是没有定义init的方法的
    RZDBCoalesceProxy *proxy = [self alloc];
    
    proxy.representedObject = object;

    return proxy;
}

- (Class)class
{
    return [self.representedObject class];
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel
{
    return [self.representedObject methodSignatureForSelector:sel];
}

- (void)forwardInvocation:(NSInvocation *)invocation
{
	if ([self.representedObject respondsToSelector:invocation.selector]) {
		invocation.target = self.representedObject;
	    [invocation invoke];
	}
}

- (void)doesNotRecognizeSelector:(SEL)aSelector
{
// 当没有实现target所指向的selector的时候，会调用到此方法
	[self.representedObject doesNotRecognizeSelector:aSelector];
}

@end
```
调用到方式，也很简单，下面简单的伪代码
```
id object = [myProxy proxyForObject: obj];
object.doSomeString()
```

#####和NSObject的区别
直接写出结论：
通过继承自`NSObject`的代理类是不会自动转发`respondsToSelector:`和`isKindOfClass:`这两个方法的, 而继承自`NSProxy`的代理类却是可以的. 测试<NSObject>中定义的其它接口二者表现都是一致的.

其实大部分使用都是一致的

大家可以参考老谭的博客：
>使用NSProxy和NSObject设计代理类的差异。http://www.tanhao.me/code/160702.html/




