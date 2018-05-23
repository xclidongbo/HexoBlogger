---
title: iOS面试题
date: 2018-05-18 19:20:18

categories:
    -iOS面试题

tags:
    -iOS

---

## iOS面试题

### 1. 多线程, 特别是NSOperation和GCD的原理.

    多线程是一个比较轻量级的方法来实现一个应用程序内多个代码执行路径.

    iPhone中的线程并不是无节制的,主线程的堆栈大小是1M,第二个线程开始就是512KB.只有主线程有修改UI的能力.

    一个运行的程序就是一个进程,一个进程至少包含一个线程.

    Mac和iOS中程序启动,创造好一个进程的同时,一个线程便开始运作,这个线程叫做主线程.

    iOS有三种线程编程技术:
        1. NSSThread
        2. NSOperation
        3. GCD

#### 1) NSThread

    优点: 比其它两个轻量级
    缺点: 需要自己管理线程的生命周期,线程同步.线程同步对数据的加锁有一定的系统开销.

#### 2) NSOperation

    优点: 不需要关心线程管理, 数据同步的事情.相关的类有NSOperation,NSOperationQueue.NSOperation是个抽象类,使用它时必须用它的子类.可以自定义或者用它定义好的两个子类: NSInvocationOperation和NSBlockOperation.

#### 3) GCD

    1. 任务: 就是所需要做的事情, 也就是用多线程执行一段代码,在GCD中要执行的任务被包含在一个函数或者block对象中,随后调用dispatch_[a]sync方法或者dispatch_[a]sync_f()方法添加到分派队列当中.
    2. 队列 队列分为串行队列和并行队列. 都是遵循先入先出(FIFO)原则.串行队列会根据任务被加入到队列的顺序,依次取出任务执行,一个任务执行完毕进行下一个任务.串行队列用于特定资源的访问(类似于锁的机制,防止共享资源的同时访问).并行队列并不等待前一个任务执行完毕才开始下一个任务.而是依次取出放入新的线程中执行.

        队列和执行顺序的说明: 
            1. 串行队列同步执行. (全在主线程中顺序执行, 循环结束后主线程的打印才输出) start 0-9 stop 都在主线程中执行
            2. 串行队列异步执行.(系统会开1个异步线程执行循环,和主线程中打印内容的先后顺序不确定) start 0-9子线程 stop的位置不定
            3. 并发队列异步执行.(异步线程和主线程的打印顺序不确定) start stop主队列 0-9多个子线程
            4. 并发队列同步执行.(因为是同步,所以是按照顺序执行和串行队列同步执行的结果一样,按照顺序执行,后面的都要等) start 0-9 stop 都在主线程中执行
            5. 主队列, 异步执行.(主线程的任务执行完毕,才能轮到主队列) start stop 0-9 都在主线程
            6. 主队列, 同步执行.(死锁,主线程中运用主队列,相当于把任务放在主线程的队列当中,而同步对于任务是立即执行的.当把任务放进主队列,它就会立即执行.但此时主线程正在运行,主队列的任务要等待主线程的任务执行完毕才能执行.主线程的任务和第一个任务就开始互相等待.)(另一种理解: 主线程的特点: 先执行主线程上的代码,后执行主队列上的代码. 同步执行dispatch_sync函数的特点: 该函数只有在添加到某队列的某方法执行完毕才会返回.即等待task执行完再返回)
            7. 全局队列. (其本质是并行队列)

        [示例查看](https://www.jianshu.com/p/8ab497e39eb5)

### 2. runtime的原理和运用场景

   runtime是比较底层的C语言API.编写的OC代码最终都转成了runtime的C语言代码.runtime最主要的是消息机制.对于C语言函数的调用在编译时候会决定调用哪个函数.OC的函数调用为消息发送.属于动态调用过程.在编译的时候并不能决定调用哪个函数.只有在运行的时候才会根据函数的名称来找到对应的函数调用.

   实例对象instance=>类class=>方法method(=>SEL=>IMP)

   runtime的运用场景:
        1. 发送消息 (消息机制原理: 对象根据方法SEL编号来查找对应的方法)
        2. 交换方法 (开发使用场景: 系统自带的方法功能不够,给系统自带的方法扩展一些功能.并保持原有的功能.)
        3. 动态添加方法 (开发使用场景: 如果一个类方法非常多, 加载类到内存的时候比较耗费资源, 需要给每个方法生成映射表, 可以使用动态给某个类,添加方法解决)
        4. 给分类添加属性 (原理: 给一个类盛名属性,其实本质就是给这个类添加关联, 并不是直接把这个值的内存空间添加到内存空间)
        5. 字典转模型 (遍历模型中所有的属性,根据模型属性名去字典中查找key,取出相应的值,给模型的属性赋值)

### 3. SDWebImage的原理?实现机制?如何解决tableView卡顿的问题?
    1. 入口`setImageWithURL:placeholderImage:options:`会把placeholderImage显示,然后`SDWebImageManager`根据URL开始处理图片.
    2. 进入`SDWebImageManager`-`downloadWithURL:delegate:options:userInfo:`,交给`SDImageCache`从缓存查找图片是否开始下载`queryDiskCacheForKey:delegate:userInfo:`.
    3. 先从内存图片缓存查找是否有图片,如果内存中已经有图片缓存, `SDImageCacheDelegate`回调`imageCache:didFindImage:forKey:userInfo:`到`SDWebImageManager`.
    4. `SDWebImageManagerDelegate`回调`webImageManager:didFinishWithImage:` 到`UIImageView+WebCache`等前端展示图片.
    5. 如果内存缓存中没有, 生成`NSInvocationOperation`添加到队列开始从磁盘查找图片是否已经缓存.
    6. 根据URLKey在硬盘缓存下尝试读取图片.这一步在`NSOperation`进行的操作,所以回主线程进行结果回调notifyDelegate:.
    7. 如果上一操作从硬盘读取到了图片,将图片添加到内存缓存中(如果空闲内存过小,会先清空缓存).`SDImageCacheDelegate`回调`imageCache:didFindImage:forKey:userInfo:`.进而回调展示图片.
    8. 如果从硬盘读取不到图片,说明所有缓存都不存在该图片, 需要下载图片. 回调`imageCache:didNotFindImage:forKey:userInfo:`.
    9. 共享或者重新生成一个下载器`SDWebImageDownloader`开始下载图片.
    10. 图片下载由`NSURLConnection`来做,实现相关代理来判断图片下载中,下载完成和下载失败.
    11. `connection:didReceiveData:`利用ImageIO来做图片下载进度的加载效果.
    12. `connectionDidFinishLoading:`数据下载完成后交给SDWebImageDecoder做图片解码处理.
    13. 图片解码处理在一个`NSOperationQueue`完成,不会拖慢主线程UI.如果需要对下载图片进行二次处理,最好也在这里完成. 效率会好很多.
    14. 在主线程`notifyDelegateOnMainThreadWithInfo:`宣告解码完成, `imageDecoder:didFinishDecodingImage:userInfo:`回调给`SDWebImageDownloader`.
    15. `imageDownloader:didFinishWithImage:`回调给SDWebImageManager告知图片下载完成.
    16. 通知所有的`downloadDelegates`下载完成,回调给需要的地方展示图片.
    17. 将图片保存到SDImageCache中, 内存缓存和硬盘缓存同时保存.写文件到硬盘也在单独的NSInvocationOperation完成,避免拖慢主线程.
    18. `SDImageCache`在初始化的时候会注册一些消息通知.在内存警告或者退到后台的时候清理内存图片缓存,应用结束的时候清理过期图片.
    19. `SDWebImage`也提供了`UIButton+WebCache`和`MKAnnotationView+WebCache`方便使用.
    20. `SDWebImagePrefetcher`可以预先下载图片, 方便后续使用.

    tableView滑动卡顿的原因是从缓存或者从本地读取图片给UIImage的时候耗费时间.可以把下面两句放在子线程:
    ```
    NSData *imgData = [NSData dataWithContentsOfURL:[NSURL URLWithString:app.icon]]; //得到图像数据  
    UIImage *image = [UIImage imageWithData:imgData];
    ```
### 4. block和代理,通知的区别. block的用法需要注意什么.
    1. Notification: 一对多
    2. delegate: 一对一.单利对象不能使用代理.代理更重视过程信息的传输,比如判断网络请求是否完成,数据接受失败.
    3. block: 一对一通信.写法更简练.block更注重结果传输.block要注意循环引用.

### 5. strong, weak, retain, assign, copy nonatomic等的区别

    - assigin: 修饰基础数据类型.不会增加引用计数.只用来声明基本的数据类型,分配到栈上,栈由系统处理,并不会造成野指针.
    - retian: 每次被引用,引用计数+1.只有当引用计数为0,才会被dealloc析构函数回收内存.
    - copy: 常见copy声明NSString. copy与retain的区别是,retian引用是拷贝指针地址,而copy是拷贝对象本身.也就是说retian是浅复制,copy是深复制.
    - weak: 类似于assign,弱引用,也不增加引用计数.一般只有的防止循环引用的时候使用.IBOutlet,Delegate一般用的是weak,因为它们会在类外部被调用,防止循环引用.
    - strong: 强引用
    - nonatomic: 非原子访问.多线程为了避免在操作时同时读写造成问题.经常要对对象进行加锁,只允许一个线程去操作它.如果一个属性由atomic修饰,那么系统就会线程保护,防止多个操作共同进行.


### 6. 设计模式, mvc, 单例, 工厂,代理等的运用场景.


### 7.单例的写法, 在单例中创建数组应该注意写什么?
    ```
    static Singleton * instance = nil;
    + (instancetype) shareInstance
    {
        static dispatch_once_t onceToken;
        dispatch_once(&onceToken, ^{
            instance = [[self alloc] init];
        });
        return instance;
    }
    ```

    单例里面添加NSMutableArray的时候, 要防止多个地方对它同时遍历和修改.需要增加原子属性, strong.并且写一个遍历和修改的方法,加锁.

### 8.NSString的时候用copy和strong的区别.

    用在`NSString`上面的话,strong和copy的结果都是一样的.而为何`NSString`要用`copy`而不是`strong`.因为,假如是strong,`NSString`的指针可以持有`NSMutableString`对象,为了防止MutableString被无意中篡改.也会导致`NSString`也会遭到修改.原则上这是不允许的.所以会用copy修饰.

### 9.响应链

    当响应事件发生时,必须支持由谁来响应事件.在iOS中,由响应者链来响应.所有事件响应的类都是UIResponder的子类. 发生事件时,事件首先发送给第一响应者.事件沿着响应者链向下传递,直到被接受并做出处理.第一响应者是个视图对象或者其子类对象,被触摸后事件交给它处理.如果不处理,事件就传给它的视图控制器ViewController.然后是它的父视图,以此类推,达到顶层视图,接着到达窗口`UIWindow`再到程序的UIApplication对象.如果整个过程没响应,就丢弃.

### 10. NSTimer在子线程中应该手动获取NSRunloop对象,否则不能循环执行.

### 11. UIScrollView和NSTimer组合做循环广告轮播的时候有一个属性可以控制上下滚动tableVIew的时候,广告轮播图依然能正常滚动.

### 12.git和svn的用法.

    推荐git来做代码同步

### 13.友盟报错可以查到具体某一行的错误, 原理是什么?

### 14.Instrument可以检测电池的耗电量,和内存的消耗的用法.


### 15. ARC的原理. 

    ARC: 自动引用计数. 消除了手动管理内存的繁琐.编译器会自动在适当的地方插入retian, release, autorelease语句.ARC是编译器特性.而不是运行时特性.
    ARC原理:
    规则: 只有一个强指针变量指向对象, 对象就会保留在内存里.默认所有的实例变量和局部变量,都是Strong指针.弱指针指向对象被回收后, 弱指针会自动变为nil指针,不会引发野指针.
    使用注意:
        - 不能调用release, retian,autorelease,retianCount.
        - 可以重写dealloc, 但不能调用[Super dealloc];
        - 想长期拥有某个对象,应该用strong,而不是weak.
        - 其他基本类型依然用assign.
        - 两端互相引用时,一端用strong,一端用weak.


