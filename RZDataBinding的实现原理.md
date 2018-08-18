####RZDataBinding的实现原理

最近面试，有人问到，`MVVM`开发模式和`RZDataBinding`实现原理，以前只是大致知道实现原理，没有仔细进行总结。下面画出半天的时间，总结一下。

下面来讲一讲它的实现原理。
它的具体用法，我们不再这里做过多的阐述，有兴趣可以在`GitHub`上，可以查看，里面写的有很详细。
大家都很清楚`RZDataBinding`是做什么的，主要是实现了动态的双向绑定，那么我们就猜想，带着问题，来叙述`RZDataBinding`是如何实现的。
1，在oc中实现绑定，第一想法是使用`KVO`，`RZDataBinding`是否使用了`KVO`
2，如何实现多值的绑定
3，如何实现绑定立即生效
4，如何做到取消绑定，取消监听

#####RZDataBinding是否使用了kvo
在我们通过`RZDataBinding`调用暴漏出来的`rz_addTarget`还是`rz_bindKey`，最终都会调用到
```
- (void)setTarget:(id)target action:(SEL)action boundKey:(NSString *)boundKey bindingTransform:(RZDBKeyBindingTransform)bindingTransform
{
    self.target = target;
    self.action = action;
    self.methodSignature = [target methodSignatureForSelector:action];

    self.boundKey = boundKey;
    self.bindingTransform = bindingTransform;

    [self.observedObject addObserver:self forKeyPath:self.keyPath options:self.observationOptions context:kRZDBKVOContext];
}
```
通过这个方法，我可以清楚看到，`RZDataBinding`也是使用`kvo`，进行监听值的改变。虽然通过`kvo`做到了监听，可是在这里有一个疑问，如何做到值的更改？
```
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context
{
    if ( context == kRZDBKVOContext ) {
        id target = nil;
        SEL action = NULL;
        NSDictionary *changeDict = nil;

        @synchronized (self) {
            target = self.target;
            action = self.action;

            if ( self.methodSignature.numberOfArguments > 2 ) {
                changeDict = [self changeDictForKVOChange:change];
            }
        }

        if ( changeDict != nil ) {
            ((void(*)(id, SEL, NSDictionary *))objc_msgSend)(target, action, changeDict);
        }
        else {
            ((void(*)(id, SEL))objc_msgSend)(target, action);
        }
    }
}
```
我们通过代码发现，它是通过`runntime`方法，`objc_msgSend`进行消息的发送，也就是值的更改。
在这里需要特别说明一下，因为作者暴漏出来两种类型的API，一种是`rz_addTarget`，一种是`rz_bindKey`，这两种更新方式，还是有一点不同。
```objectivec
- (void)rz_addTarget:(id)target action:(SEL)action forKeyPathChange:(NSString *)keyPath;
```
在前面调用`objc_msgSend`，调用的是`target`中实现的`action`，我们可以理解为，是使用者自己的类的实现方法。但是`rz_bindKey`不同
```objectivec
- (void)rz_bindKey:(NSString *)key toKeyPath:(NSString *)foreignKeyPath ofObject:(id)object;
```
在此API中，我们没有看到类似`rz_addTarget`中的`action`，那么它是如何做到更新的。首先，把结论告诉大家，也许猜到，使用的是`kvc`，下面我们看如何具体实现：
在我们调用`rz_bindKey`的时候，在内部依然调用的`rz_addTarget`
```objectivec
- (void)rz_bindKey:(NSString *)key toKeyPath:(NSString *)foreignKeyPath ofObject:(id)object withTransform:(RZDBKeyBindingTransform)bindingTransform
{
    NSParameterAssert(key);
    NSParameterAssert(foreignKeyPath);
    
    if ( object != nil ) {
        @try {
            id val = [object valueForKeyPath:foreignKeyPath];

            [self rz_setBoundKey:key withValue:val transform:bindingTransform];
        }
        @catch (NSException *exception) {
            [NSException raise:NSInvalidArgumentException format:@"RZDataBinding cannot bind key:%@ to key path:%@ of object:%@. Reason: %@", key, foreignKeyPath, [object description], exception.reason];
        }
        
        [object rz_addTarget:self action:@selector(rz_observeBoundKeyChange:) boundKey:key bindingTransform:bindingTransform forKeyPath:foreignKeyPath withOptions:NSKeyValueObservingOptionNew];
    }
}
```
在上面的代码中，我看到`rz_observeBoundKeyChange`,在bind的key改变之后，调用就是此方法，我们看一下此方法的实现：
```
- (void)rz_observeBoundKeyChange:(NSDictionary *)change
{
    NSString *boundKey = change[kRZDBChangeKeyBoundKey];
    
    if ( boundKey != nil ) {
        id value = change[kRZDBChangeKeyNew];

        [self rz_setBoundKey:boundKey withValue:value transform:change[kRZDBChangeKeyBindingTransformKey]];
    }
}

- (void)rz_setBoundKey:(NSString *)key withValue:(id)value transform:(RZDBKeyBindingTransform)transform
{
    id currentValue = [self valueForKey:key];

    if ( transform != nil ) {
        value = transform(value);
    }

    if ( currentValue != value && ![currentValue isEqual:value] ) {
        [self setValue:value forKey:key];
    }
}
```
我们看到了调用了`setValue`方法

#####如何实现多值的绑定
上面介绍了，当监听一个value更改的时候的操作，我们在实际的工作场景中，遇到很多，当多个值，其中一个值进行更改的时候，ui上需要作出一些更新或者其他操作，在`RZDataBinding`中，同时也提供了此API，我们以`rz_addTarget`为例
```objectivec
- (void)rz_addTarget:(id)target action:(SEL)action forKeyPathChanges:(NSArray *)keyPaths;
```
其实很简单，就是做了一个循环
```objectivec
[keyPaths enumerateObjectsUsingBlock:^(NSString *keyPath, NSUInteger idx, BOOL *stop) {
        [self rz_addTarget:target action:action forKeyPathChange:keyPath callImmediately:callMultiple];
    }];
```
#####如何实现绑定立即生效
其实，依然很简单，直接调用了`objc_msgSend`
```
if ( callImmediately && !callMultiple ) {
        ((void(*)(id, SEL))objc_msgSend)(target, action);
    }
```

#####如何做到取消绑定，取消监听
在我们的开发工作中，大部分是使用`ARC`去管理内存，我们不需要主动的去释放，但是，在我们使用通知，或者监听的时候，依然需要`remove`操作，`RZDataBinding`是如何做到释放的呢，在它提供的API中，我们也看到了，它提供了各种remove的操作，但是，如何做到自动取消绑定和监听的呢？首先说出结论：使用`runntime`中的，方法的转换，我们俗称的黑魔法，对`dealloc`方法的实现的转换.

在讲如何实现之前，我们首先讲一种宏定义，我会其他篇幅里，仔细去讲，在这里只是提一下
```objectivec
#ifndef RZDB_AUTOMATIC_CLEANUP
#define RZDB_AUTOMATIC_CLEANUP 1
#endif
```
表达的含义：当`RZDB_AUTOMATIC_CLEANUP`没有定义，就是执行`#define RZDB_AUTOMATIC_CLEANUP 1`

下面我们继续看代码
```objectivec
#if RZDB_AUTOMATIC_CLEANUP
    rz_swizzleDeallocIfNeeded([self class]);
    rz_swizzleDeallocIfNeeded([target class]);
#endif
```
通过此代码，当`RZDB_AUTOMATIC_CLEANUP`定义的时候，也就是会自动进行实行，会执行`rz_swizzleDeallocIfNeeded`，通过名字，我们可以看到，对`dealloc`的`swizzle`，下面我们看一下具体的实现
```objectivec
void rz_swizzleDeallocIfNeeded(Class class)
{
    static SEL deallocSEL = NULL;
    static SEL cleanupSEL = NULL;

    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        deallocSEL = sel_getUid("dealloc");
        cleanupSEL = sel_getUid("rz_cleanupObservers");
    });

    @synchronized (class) {
        if ( !rz_requiresDeallocSwizzle(class) ) {
            // dealloc swizzling already resolved
            return;
        }

        objc_setAssociatedObject(class, kRZDBSwizzledDeallocKey, @(YES), OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }

    Method dealloc = NULL;

    // search instance methods of the class (does not search superclass methods)
    unsigned int n;
    Method *methods = class_copyMethodList(class, &n);

    for ( unsigned int i = 0; i < n; i++ ) {
        if ( method_getName(methods[i]) == deallocSEL ) {
            dealloc = methods[i];
            break;
        }
    }

    free(methods);

    if ( dealloc == NULL ) {
        Class superclass = class_getSuperclass(class);

        // class does not implement dealloc, so implement it directly
        class_addMethod(class, deallocSEL, imp_implementationWithBlock(^(__unsafe_unretained id self) {

            // cleanup RZDB observers
            ((void(*)(id, SEL))objc_msgSend)(self, cleanupSEL);

            // ARC automatically calls super when dealloc is implemented in code,
            // but when provided our own dealloc IMP we have to call through to super manually
            struct objc_super superStruct = (struct objc_super){ self, superclass };
            ((void (*)(struct objc_super*, SEL))objc_msgSendSuper)(&superStruct, deallocSEL);

        }), method_getTypeEncoding(dealloc));
    }
    else {
        // class implements dealloc, so extend the existing implementation
        __block IMP deallocIMP = method_setImplementation(dealloc, imp_implementationWithBlock(^(__unsafe_unretained id self) {
            // cleanup RZDB observers
            ((void(*)(id, SEL))objc_msgSend)(self, cleanupSEL);

            // invoke the original dealloc IMP
            ((void(*)(id, SEL))deallocIMP)(self, deallocSEL);
        }));
    }
}
```
通过阅读这个方法 代码，核心逻辑：实现判断class的dealloc的方法，是否实现，`if dealloc == null`通过runntime的`class_addMethod`，动态增加一个回调，向`cleanupSEL`发送消息。`if dealloc != null`，动态改变了实现`method_setImplementation`，在此回调中，往`cleanupSEL`和`deallocSEL`发送消息。在此解决了，自动释放的效果。

>备注：在`RZDataBinding`代码中，还有通过`NSProxy`，实现了消息的集体发送，同时也是使用代理，实现了一种消息转发，下个篇章，我们来进行介绍。



