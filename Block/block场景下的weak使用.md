#Block场景下的weak使用
在讲解正文之前，先说一个block修饰符为copy的缘由。在[苹果官方文档](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/WorkingwithBlocks/WorkingwithBlocks.html#//apple_ref/doc/uid/TP40011210-CH8-SW12)中有官方示例，如下所示：

>@interface XYZObject : NSObject

>@property (copy) void (^blockProperty)(void);

>@end


Note: You should specify copy as the property attribute,    because a block needs to be copied to keep track of its captured state outside of the original scope. This isn’t something you need to worry about when using Automatic Reference Counting, as it will happen automatically, but it’s best practice for the property attribute to show the resultant behavior. 


block 使用 copy 是从 MRC 遗留下来的“传统”。在 MRC 中,方法内部的 block 是在栈区的,使用 copy 可以把它放到堆区。在 ARC 中写不写都行：对于 block 使用 copy 还是 strong 效果是一样的，但写上 copy 也无伤大雅，还能时刻提醒我们：编译器自动对 block 进行了 copy 操作。如果不写 copy ，该类的调用者有可能会忘记或者根本不知道“编译器会自动对 block 进行了 copy 操作”，他们有可能会在调用之前自行拷贝属性值。这种操作多余而低效。你也许会感觉我这种做法有些怪异，不需要写依然写。如果你这样想，其实是你“日用而不知”，你平时开发中是经常在用我说的这种做法的，比如下面的属性不写copy也行，但是你会选择写还是不写呢？

###一、Block场景分析
在实际使用Block过程中，经常会遇到各种问题，比如**Retain Cycle**，这时候大家可能第一个想到的就是使用**__weak**修饰对象打破循环引用。在此之前我们需要明确一点：**是不是所有的block，使用self都会出现循环引用？** 其实不然，系统和第三方框架的block绝大部分不会出现循环引用，只有少数block以及我们自定义的block会出现循环引用。在使用**__weak**之前，要详细的分析是否造成了循环引用。下面我们分析几种场景：

-  UIViewAnimationWithBlocks
-  GCD
-  enumerateObjectsUsingBlock
-  MJRefreshHeader
-  AFNetworking

####1. UIViewAnimationWithBlocks
    [UIView animateWithDuration:0.25 animations:^{  
       self.testView.frame = CGRectMake(80, 100, 100,100); 
    }];
这里是可以直接写self的，因为这是一个类方法，当前的self并没有直接或间接持有这个block。不会循环引用 
> animation framework -> block
>
> block -> self

####2. GCD

>dispatch确实产生了self->_queue->block->self循环引用，但queue在事后主动释放了block，破除了循环引用。

>在YYCache源码中，ibireme大神在_trimInBackground方法中使用dispatch_async用到了__weak，ibireme 认为self->_queue->block->self构成了循环引用。具体源码如下：

    - (void)_trimInBackground {
        __weak typeof(self) _self = self;
        dispatch_async(_queue, ^{
            __strong typeof(_self) self = _self; // ①
            if (!self) return; // ②
            dispatch_semaphore_wait(self->_lock, DISPATCH_TIME_FOREVER);
            [self _trimToCost:self.costLimit];
            [self _trimToCount:self.countLimit];
            [self _trimToAge:self.ageLimit];
            [self _trimToFreeDiskSpace:self.freeDiskSpaceLimit];
            dispatch_semaphore_signal(self->_lock);
        });
    }
但笔者认为dispatch时，self->_queue->block->self确实产生了循环引用，但queue在事后主动释放了block，破除了循环引用。所以此处也可以不用__Weak破除循环。


####3. enumerateObjectsUsingBlock
NSArray的enumerateObjectsUsingBlock 跟UIViewAnimationWithBlocks同一个道理self并不会持有block, 所以不会循环引用。

    [self.dataArray enumerateObjectsUsingBlock:^(NSString *str, NSUInteger idx, BOOL * _Nonnull stop) {
        [self dosomething:str];
    }];
    
 > Foundation framework-> block
 
 > block -> self

####4. MJRefreshHeader
    //下拉刷新
    WEAKSELF;
    self.tableView.mj_header = [MJRefreshNormalHeader headerWithRefreshingBlock:^{
        [weakSelf dosomething];
    }];
这里用了**__weak**,我们来具体分析一下：

> self -> tableView -> mj_header.block -> self

####5. AFNetworking
    AFHTTPSessionManager *session = [AFHTTPSessionManager manager];
    [session GET:testURL parameters:nil progress:nil success:^(NSURLSessionDataTask * _Nonnull task, id  _Nullable responseObject) {
        NSLog(@"请求成功---%@", responseObject);
        self.testLabel.text = @"请求成功";
    } failure:^(NSURLSessionDataTask * _Nullable task, NSError * _Nonnull error) {
        NSLog(@"%@",error);
    }];
    
具体分析如下：

>AFHTTPSessionManager -> AFURLSessionManagerTaskDelegate(包含manager)
>
>AFURLSessionManagerTaskDelegate ->ViewController block

未发生循环引用问题 所以可以直接使用self

###二、Block引用计数测试
在实际使用中，当遇到无法确定是否发生循环引用的，可以通过实际测试确定。

####条件1:MRC + copy
    @property (nonatomic,copy) GXBlock blockTest;
    
    - (void)test {
        
        NSLog(@"1.self的引用计数=%ld",CFGetRetainCount((__bridge CFTypeRef)(self)));  //1
        self.blockTest = ^{
            NSLog(@"self == %@",self);
        };
        
        self.blockTest();
        
        NSLog(@"1.block的引用计数=%ld",CFGetRetainCount((__bridge CFTypeRef)(self.blockTest))); //1
        NSLog(@"2.self的引用计数=%ld",CFGetRetainCount((__bridge CFTypeRef)(self)));   //2
        
        [self.blockTest copy];
        NSLog(@"1.block的引用计数=%ld",CFGetRetainCount((__bridge CFTypeRef)(self.blockTest))); //1
        NSLog(@"3.self的引用计数=%ld",CFGetRetainCount((__bridge CFTypeRef)(self)));   //2
        [self.blockTest copy];
        NSLog(@"4.block的引用计数=%ld",CFGetRetainCount((__bridge CFTypeRef)(self.blockTest))); //1
        NSLog(@"5.self的引用计数=%ld",CFGetRetainCount((__bridge CFTypeRef)(self)));   //2
        
        [self.blockTest copy];
        NSLog(@"6.block的引用计数=%ld",CFGetRetainCount((__bridge CFTypeRef)(self.blockTest))); //1
        NSLog(@"7.self的引用计数=%ld",CFGetRetainCount((__bridge CFTypeRef)(self)));   //2
        
        
        [self release];
        NSLog(@"8.self的引用计数=%ld",CFGetRetainCount((__bridge CFTypeRef)(self))); //1
    }
测试结果如下：  
![MRC + copy](https://ws3.sinaimg.cn/large/006tNc79ly1g2sjc7gto6j30og06279u.jpg)   

####条件2：MRC + assign
     @property (nonatomic,assign) GXBlock blockTest;
![MRC + assign](https://ws4.sinaimg.cn/large/006tNc79ly1g2sjd672xyj30qj05y0y0.jpg) 

####条件3：ARC + copy
![ARC + copy](https://ws1.sinaimg.cn/large/006tNc79ly1g2sjdq1gvaj30nv04vjv4.jpg) 

####条件4：ARC + assign
![ARC + assign](https://ws1.sinaimg.cn/large/006tNc79ly1g2sjefu9q0j313q094tdj.jpg) 

###小结：
1. 拷贝堆上的block ，self 引用计数不会改变。拷贝栈上的block，self的引用计数+1；  
2. ARC机制下，会自动拷贝栈上的block至堆上，所以手动调用copy时，不会增加self的引用计数；MRC下，每次将栈上的block拷贝至堆后，self引用计数+1；
3. assign修饰block不会增加引用计数，copy修饰block会在增加引用计数。
4. ARC下，copy修饰的block中有self时，self引用计数+2；assign修饰的block中有self时，引用计数+1；
5. MRC下，copy修饰的block中有self时，self引用计数+1；assign修饰的block中有self时，引用计数不变；（正确的）
6. 使用weakself可以破除循环引用的问题；


