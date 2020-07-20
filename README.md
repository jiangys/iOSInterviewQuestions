## iOSInterviewQuestions
生成索引目录：https://github.com/ekalinin/github-markdown-toc  下载下来，打开gh-md-toc所有的目录，输入./gh-md-toc /Users/yongsheng/VSMVVM/README.md即可

## 索引
  * [KVO的原理](#kvo的原理)
      * [block 里面的变量为什么要用copy](#block-里面的变量为什么要用copy)
      * [weak原理](#weak原理)
      * [离屏渲染怎么产生，怎么避免](#离屏渲染怎么产生怎么避免)
      * [tableView性能优化](#tableview性能优化)
## 题解
### 一个NSObject对象占用多少内存？
系统分配了16个字节给NSObject对象（通过malloc_size函数获得）  
但NSObject对象内部只使用了8个字节的空间（64bit环境下，可以通过class_getInstanceSize函数获得）
在源代码中有对齐逻辑，如果字节小于8，会自动补齐到

### 对象的isa指针指向哪里？
- instance对象的isa指向class对象
- class对象的isa指向meta-class对象
- meta-class对象的isa指向基类的meta-class对象

### OC的类信息存放在哪里？
- 对象方法、属性、成员变量、协议信息，存放在class对象中
- 类方法，存放在meta-class对象中
- 成员变量的具体值，存放在instance对象

### KVO的原理是什么？(KVO的本质是什么？)
KVO的全称是Key-Value Observing，俗称“键值监听”，可以用于监听某个对象属性值的改变。  
原理: 利用RuntimeAPI动态生成一个子类，并且让instance实例对象的isa指向这个全新的子类（如：NSKVONotifying_YSPerson），当修改instance实例对象的属性时，会调用Foundation的_NSSetXXXValueAndNotify函数，该函数的内部实现如下
1. willChangeValueForKey:
2. 父类原来的setter
3. didChangeValueForKey:内部会触发监听器（Oberser）的监听方法( observeValueForKeyPath:ofObject:change:context:）   
可以通过写【- (void)willChangeValueForKey:】方法来验证，会调用。当然手动添加上面的流程，也会触发KVO的方法调用。   
直接修改成员变量会触发KVO吗？不会   
通过KVC修改属性会触发KVO么? 会触发
手动触发KVO
```Objective-C
    [self.person willChangeValueForKey:@"age"];
    [self.person didChangeValueForKey:@"age"];
 ```
### KVC的赋值和取值过程是怎样的？原理是什么？
KVC（Key-value coding）键值编码。简单来说指iOS的开发中，可以允许开发者通过Key名直接访问对象的属性，或者给对象的属性赋值，而不需要调用明确的存取方法。这样就可以在运行时动态地访问和修改对象的属性。是iOS开发中的黑魔法之一，很多高级的iOS开发技巧都是基于KVC实现的。
运用场景：
1. 动态地设值和取值
2. 用KVC来访问和修改私有变量
3. model和字典互转 
4. 修改一些系统控件的内部属性，使用runtime来获取Apple不想开放的成员变量，利用KVC进行修改，比如自定义tabbar，textfield等，这个的应用也是比较常见

<img src="https://img2020.cnblogs.com/blog/292326/202007/292326-20200716151500784-1588098047.png" width="700"><br/>

### Category的实现原理
1. 通过Runtime加载某个类的所有Category数据,Category编译之后的底层结构是struct category_t，里面存储着分类的对象方法、类方法、属性、协议信息
2. 在程序运行的时候，runtime会将Category的数据，合并到类信息中（类对象、元类对象中）
定义在objc-runtime-new.h中
图片1
### Category如何添加成员变量？
#### 添加关联对象
```Objective-C
// 添加关联对象  
void objc_setAssociatedObject(id object, const void * key,id value, objc_AssociationPolicy policy)
// 获得关联对象   
id objc_getAssociatedObject(id object, const void * key)
// 移除所有的关联对象  
id objc_getAssociatedObject(id object, const void * key)
```
#### 常用用法：
```Objective-C
static void *MyKey = &MyKey;
objc_setAssociatedObject(obj, MyKey, value, OBJC_ASSOCIATION_RETAIN_NONATOMIC)
objc_getAssociatedObject(obj, MyKey)

// 使用get方法的@selecor作为key
objc_setAssociatedObject(obj, @selector(getter), value, OBJC_ASSOCIATION_RETAIN_NONATOMIC)
objc_getAssociatedObject(obj, @selector(getter))
```
#### 关联对象原理
关联对象并不是存储在被关联对象本身内存中，而是存储在全局的统一的一个AssociationsManager中，里面是由HashMap来管理。  
实现关联对象技术的核心对象有（objc4源码解读：objc-references.mm）
- AssociationsManager
- AssociationsHashMap
- ObjectAssociationMap
- ObjcAssociation

<img src="https://img2020.cnblogs.com/blog/292326/202007/292326-20200716151620317-1773536446.png" width="700"><br/>

#### +initialize和+load的的区别
##### +load方法
会在runtime加载类、分类时调用，是根据方法地址直接调用，并不是经过objc_msgSend函数调用  
每个类、分类的+load，在程序运行过程中只调用一次

**调用顺序**
1. 先调用类的+load，按照编译先后顺序调用（先编译，先调用），调用子类的+load之前会先调用父类的+load
2. 再调用分类的+load，按照编译先后顺序调用（先编译，先调用）
##### +initialize方法
会在类第一次接收到消息时调用，是通过objc_msgSend进行调用  
**调用顺序**
先调用父类的+initialize，再调用子类的+initialize
(先初始化父类，再初始化子类，每个类只会初始化1次)

### block为什么要用copy修饰
block 使用 copy 是从 MRC 遗留下来的“传统”,在 MRC 中,方法内部的 block 是在栈区的,使用 copy 可以把它放到堆区.
在 ARC 中写不写都行：
在 ARC 环境下，编译器会根据情況自动将栈上的 block 复制到堆上，比如以下情况：
- block 作为函数返回值时
- 将 block 赋值给 __strong 指针时（property 的 copy 属性对应的是这一条）
- block 作为 Cocoa API 中方法名含有 using Block 的方法参数时
- block 作为 GCD API 的方法参数时

  ![](https://tva1.sinaimg.cn/large/007S8ZIlly1gfj47m0v1wj30s01cak0r.jpg)
  
其中， block 的 property 设置为 copy， 对应的是这一条：将 block 赋值给 __strong 指针时。

  
  换句话说：

  对于 block 使用 copy 还是 strong 效果是一样的，但写上 copy 也无伤大雅，还能时刻提醒我们：编译器自动对 block 进行了 copy 操作。如果不写 copy ，该类的调用者有可能会忘记或者根本不知道“编译器会自动对 block 进行了 copy 操作”，他们有可能会在调用之前自行拷贝属性值。

### weak属性的实现原理
Runtime维护了一个weak表，用于存储指向某个对象的所有weak指针。weak表其实是一个hash（哈希）表，Key是所指对象的地址，Value是weak指针的地址（这个地址的值是所指对象的地址）数组  
weak编译解析

首先需要看一下weak编译之后具体出现什么样的变化，通过Clang的方法把weak编译成C++
```Objective-C
 NSObject *obj = ((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("alloc")), sel_registerName("init"));
 id __attribute__((objc_ownership(weak))) obj1 = obj;
```
编译之后的weak，通过objc_ownership(weak)实现weak方法，objc_ownership字面意思是：获得对象的所有权，是对对象weak的初始化的一个操作。   
weak是有Runtime维护的weak表   
在runtime源码中，可以找到'objc-weak.h'和‘objc-weak.mm’文件，并且在objc-weak.h文件中关于定义weak表的结构体以及相关的方法   
weak_table_t是一个全局weak 引用的表，使用不定类型对象的地址作为 key，用 weak_entry_t 类型结构体对象作为 value 。其中的 weak_entries 成员
```Objective-C
/**
 * The global weak references table. Stores object ids as keys,
 * and weak_entry_t structs as their values.
 */
struct weak_table_t {
    weak_entry_t *weak_entries; //保存了所有指向指定对象的weak指针   weak_entries的对象
    size_t    num_entries;              // weak对象的存储空间
    uintptr_t mask;                      //参与判断引用计数辅助量
    uintptr_t max_hash_displacement;    //hash key 最大偏移值
};
```
weak全局表中的存储weak定义的对象的表结构weak_entry_t，weak_entry_t是存储在弱引用表中的一个内部结构体，它负责维护和存储指向一个对象的所有弱引用hash表。

那么 runtime 如何实现 weak 变量的自动置nil？  
runtime 对注册的类， 会进行布局，对于 weak 对象会放入一个 hash 表中。 用 weak 指向的对象内存地址作为 key，当此对象的引用计数为0的时候会 dealloc，假如 weak 指向的对象内存地址是a，那么就会以a为键， 在这个 weak 表中搜索，找到所有以a为键的 weak 对象，从而设置为 nil。  
这需要对对象整个释放过程了解，如下是对象释放的整体流程：
1. 调用objc_release
2. 因为对象的引用计数为0，所以执行dealloc
3. 在dealloc中，调用了_objc_rootDealloc函数
4. 在_objc_rootDealloc中，调用了object_dispose函数
5. 调用objc_destructInstance
6. 最后调用objc_clear_deallocating。

可参阅[浅谈iOS之weak底层实现原理](https://www.jianshu.com/p/f331bd5ce8f8)

### 什么是runtime,平时项目中有用过么？
OC是一门动态性比较强的编程语言，允许很多操作推迟到程序运行时再进行,OC的动态性就是由Runtime来支撑和实现的，Runtime是一套C语言的API，封装了很多动态性相关的函数。平时编写的OC代码，底层都是转换成了Runtime API进行调用。

具体应用
1. 发送消息
2. 交换方法实现（交换系统的方法）
3. 利用关联对象（AssociatedObject）给分类添加属性，给alertView添加传值
4. 遍历类的所有成员变量（修改textfield的占位文字颜色、字典转模型、自动归档解档）
5. 动态添加方法
6. 字典转模型KVC实现

### 讲一下 OC 的消息机制
OC中的方法调用其实都是转成了objc_msgSend函数的调用，给receiver（方法调用者）发送了一条消息（selector方法名）

objc_msgSend底层有3大阶段

消息发送（当前类、父类中查找）、动态方法解析、消息转发

### RunLoop内部实现逻辑
RunLoop 顾名思义是运行循环，在程序运行过程中循环做一些事情。

RunLoop的基本作用
- 保持程序的持续运行
- 处理App中的各种事件（比如触摸事件、定时器事件等）
- 节省CPU资源，提高程序性能：该做事时做事，该休息时休息

RunLoop与线程
- 每条线程都有唯一的一个与之对应的RunLoop对象
- RunLoop保存在一个全局的Dictionary里，线程作为key，RunLoop作为value
- 线程刚创建时并没有RunLoop对象，RunLoop会在第一次获取它时创建
- RunLoop会在线程结束时销毁
- 主线程的RunLoop已经自动获取（创建），子线程默认没有开启RunLoop

RunLoop的运行逻辑 
1. 通知Observers：进入Loop
2. 通知Observers：即将处理Timers
3. 通知Observers：即将处理Sources
4. 处理Blocks
5. 处理Source0（可能会再次处理Blocks）
6. 如果存在Source1，就跳转到第8步
7. 通知Observers：开始休眠（等待消息唤醒）
8. 通知Observers：结束休眠（被某个消息唤醒）
- 处理Timer
- 处理GCD Async To Main Queue
- 处理Source1
9. 处理Blocks
10. 根据前面的执行结果，决定如何操作
- 回到第02步
- 退出Loop
11. 通知Observers：退出Loop

RunLoop休眠的实现原理   
有个用户态和内核态，mach_msg()，等待消息
- 没有消息就让线程休眠
- 有消息就唤醒线程


### RunLoop在实际开中的应用
- 控制线程生命周期（线程保活）
- 解决NSTimer在滑动时停止工作的问题
- 监控应用卡顿
- 性能优化

### 多线程锁有多少种，分别怎么使用
NSRecursiveLock实际上定义的是一个递归锁，这个锁可以被同一线程多次请求，而不会引起死锁。这主要是用在循环或递归操作中
```Objective-C
NSLock *lock = [[NSLock alloc] init];
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    static void (^RecursiveMethod)(int);
    RecursiveMethod = ^(int value) {
        [lock lock];
        if (value > 0) {
            NSLog(@"value = %d", value);
            sleep(2);
            RecursiveMethod(value - 1);
        }
        [lock unlock];
    };
    RecursiveMethod(5);
});
```
这段代码是一个典型的死锁情况。在我们的线程中，RecursiveMethod是递归调用的。所以每次进入这个block时，都会去加一次锁，而从第二次开始，由于锁已经被使用了且没有解锁，所以它需要等待锁被解除，这样就导致了死锁，线程被阻塞住了。

在这种情况下，我们就可以使用NSRecursiveLock。它可以允许同一线程多次加锁，而不会造成死锁。递归锁会跟踪它被lock的次数。每次成功的lock都必须平衡调用unlock操作。只有所有达到这种平衡，锁最后才能被释放，以供其它线程使用。

所以，对上面的代码进行一下改造，
```Objective-C
NSRecursiveLock *lock = [[NSRecursiveLock alloc] init];
```
这样，程序就能正常运行了。


### OC对象的内存管理
在iOS中，使用引用计数来管理OC对象的内存

一个新创建的OC对象引用计数默认是1，当引用计数减为0，OC对象就会销毁，释放其占用的内存空间

调用retain会让OC对象的引用计数+1，调用release会让OC对象的引用计数-1

内存管理的经验总结
当调用alloc、new、copy、mutableCopy方法返回了一个对象，在不需要这个对象时，要调用release或者autorelease来释放它
想拥有某个对象，就让它的引用计数+1；不想再拥有某个对象，就让它的引用计数-1

可以通过以下私有函数来查看自动释放池的情况  
extern void _objc_autoreleasePoolPrint(void);

### APP启动流程
APP的启动可以分为2种
- 冷启动（Cold Launch）：从零开始启动APP
- 热启动（Warm Launch）：APP已经在内存中，在后台存活着，再次点击图标启动APP

APP启动时间的优化，主要是针对冷启动进行优化

通过添加环境变量可以打印出APP的启动时间分析（Edit scheme -> Run -> Arguments）  
DYLD_PRINT_STATISTICS设置为1   
如果需要更详细的信息，那就将DYLD_PRINT_STATISTICS_DETAILS设置为1  

APP的冷启动可以概括为3大阶段
1. dyld：dyld（dynamic link editor），Apple的动态链接器，可以用来装载Mach-O文件（可执行文件、动态库等）
- 装载APP的可执行文件，同时会递归加载所有依赖的动态库
- 当dyld把可执行文件、动态库都装载完毕后，会通知Runtime进行下一步的处理

2. runtime。  
由runtime负责加载成objc定义的结构，runtime所做的事情有
- 调用map_images进行可执行文件内容的解析和处理
- 在load_images中调用call_load_methods，调用所有Class和Category的+load方法
- 进行各种objc结构的初始化（注册Objc类、初始化类对象等等）
- 调用C++静态初始化器和__attribute__((constructor))修饰的函数

到此为止，可执行文件和动态库中所有的符号(Class，Protocol，Selector，IMP，…)都已经按格式成功加载到内存中，被runtime 所管理

3. main  
所有初始化工作结束后，dyld就会调用main函数
接下来就是UIApplicationMain函数，AppDelegate的application:didFinishLaunchingWithOptions:方法

### APP的启动优化
按照不同的阶段进行优化
1. dyld
- 减少动态库、合并一些动态库（定期清理不必要的动态库）
- 减少Objc类、分类的数量、减少Selector数量（定期清理不必要的类、分类）
- 减少C++虚函数数量
- Swift尽量使用struct

2. runtime
- 用+initialize方法和dispatch_once取代所有的__attribute__((constructor))、C++静态构造器、ObjC的+load

3. main
- 在不影响用户体验的前提下，尽可能将一些操作延迟，不要全部都放在finishLaunching方法中
- 按需加载

### APP安装包瘦身
安装包（IPA）主要由可执行文件、资源组成

1. 资源（图片、音频、视频等）
- 采取无损压缩
- 去除没有用到的资源： https://github.com/tinymind/LSUnusedResources

2. 可执行文件瘦身
- 编译器优化，Strip Linked Product、Make Strings Read-Only、Symbols Hidden by Default设置为YES
- 去掉异常支持，Enable C++ Exceptions、Enable Objective-C Exceptions设置为NO， Other C Flags添加-fno-exceptions
- 利用AppCode（https://www.jetbrains.com/objc/）检测未使用的代码：菜单栏 -> Code -> Inspect Code
-编写LLVM插件检测出重复代码、未被调用的代码


### 离屏渲染怎么产生，怎么避免
在OpenGL中，GPU有2种渲染方式
- On-Screen Rendering：当前屏幕渲染，在当前用于显示的屏幕缓冲区进行渲染操作
- Off-Screen Rendering：离屏渲染，在当前屏幕缓冲区以外新开辟一个缓冲区进行渲染操作

离屏渲染消耗性能的原因
- 需要创建新的缓冲区
- 离屏渲染的整个过程，需要多次切换上下文环境，先是从当前屏幕（On-Screen）切换到离屏（Off-Screen）；等到离屏渲染结束以后，将离屏缓冲区的渲染结果显示到屏幕上，又需要将上下文环境从离屏切换到当前屏幕

哪些操作会触发离屏渲染？
- 光栅化：layer.shouldRasterize = YES
- 遮罩：layer.mask
- 圆角：同时设置layer.masksToBounds = YES、layer.cornerRadius大于0。考虑通过CoreGraphics绘制裁剪圆角，或者叫美工提供圆角图片
- 阴影：layer.shadowXXX，如果设置了layer.shadowPath就不会产生离屏渲染

### tableView性能优化
可参阅[性能优化与卡顿监控](https://www.cnblogs.com/jys509/p/13296128.html)


### iOS14 新特性
1. 汉化内容完善。在之前的Beta 1中，系统内还是很多新的功能名称依旧是英文状态，如APP 资源库就是“APP Library”。
2. 新增【文件】桌面组件。新增了【文件】桌面组件，并且有2X2和2X4两种尺寸，可以显示最近编辑的内容。
3. 新增Apple music触觉反馈.新增了Apple Music触觉反馈，现在摸上去会震动。  
4. 【日历】【时钟】图标调整。日历图标从原来的“星期几”改成了“周几”，时钟的时针和分针变得更粗一些。
5. 天气小组件文字调整。在原来的天气挂件中用H、L表示最高温和最低温，更新后直接用文字显示“最高、最低”。
6. 调整第三方组件宽度。在之前的版本中，如果添加第三方小组件，那么宽度比系统默认的一些小组件更宽，新版本将第三方小组件的宽度与系统组件统一，看起来更舒服。
7. 充电图标变大。充电时闪电图标略微变大。

技术上更新：

1、App Clip
App Clip是苹果公司推出的小程序功能，该功能是基于卡片的快速应用程序，可让你在需要时访问应用程序的一小部分，而无需用户安装完整的应用程序。

定位：lightweight（轻量）、native（原生）、fast（快速）、focused（聚焦）、in-the-moment experience（瞬间体验）；  
限制：NFC、二维码、Safari、苹果iMessage、Siri建议、苹果地图等入口用户主动发起并由系统调起，App没有能力主动调起Clip；   
开发：   
- Clip需依附于主App，大小限制在10M以内，不能单独发布，发布流程和App一样，需要在itunesconnect创建一个版本并提交苹果审核；
- 入口是一段遵从Universal link格式的URL，可带参数，并可根据参数在卡片中展示不一样的标题和图片（需在itunesconnect配置）；
- 支持的框架跟App一致，例如：UIKit、SwiftUI等，限制是不能使用一些高级应用能力（追踪、后台运行等）和访问某些用户隐私信息，但可以在Clip打开的8小时内做免申请的通知和一次位置申请，后续可再弹框申请，另外Clip申请的授权可以同步到App的；  
应用：   
- 快速体验；
- 精准营销；
- 引流

2、隐私
近几届WWDC苹果对隐私问题是越来越重视，这届也提升到了一个新的高度
隐私披露
今年发布的最重要的变化是新的 app 隐私披露提示。苹果称，新的 UI 系统会以可见的方式让用户知道每个 app 收集了哪些种类的数据。此外，每个 app 都必须详细披露用来追踪用户的每种数据类型。
苹果称新的隐私披露展示系统会加入所有的应用商店中，而不仅仅是 iOS app。披露要求在每个 app 页面上明确可见，所以用户能够做出决定，避免通过应用通过数据追踪用户。
跨App追踪权限
苹果称参与跨 app 追踪的应用现在开始也需要向用户请求特殊的权限。

3、粘贴板  
iOS14在App读取剪切板时都会以下图Toast方式提示用户，以后使用剪切板做用户行为跟踪要谨慎。

4、麦克风、摄像头访问灯  
除了上面防止用户被追踪的特征外，苹果公司还增加了一个特征来应对 App 悄悄访问 iOS 设备的摄像头和麦克风的问题。指示灯

5、桌面小组件（Widgets）  
核心：实用和聚焦
屏幕小组件（Widgets）是这次 iOS 14 更新当中的重要功能之一，屏幕小组件赋予了 App 全新的入口，更丰富的交互层级，更多样的操作模式。

6、SwiftUI  
SwiftUI推出的目的之一就是作为苹果生态大一统的UI技术，来帮助开发者在越来越多的苹果设备上实现统一的开发体验的。去年在实践中还有很多值得诟病的地方，今年已经占据了主导地位，widget / app clisp / tvOS / WatchOS 都在安利使用SwiftUI；
但实际上SwiftUI没有什么颠覆性的更新，主要是更新以下方面：
- 提高稳定性；
- 新增一些新API；
- 简化语法；

7、其他  
1. 人体关键点检测技术；

参考
[苹果 App Clip 技术详解](https://mp.weixin.qq.com/s/YirFmXaY-F1V62ihbYJCuw)
[WWDC 2020 发布的安全和隐私新特征](https://www.chainnews.com/articles/377441009749.htm)

# Author
jiangys, jys509@126.com

# License
iOSInterviewQuestions is available under the MIT license. See the LICENSE file for more info.
