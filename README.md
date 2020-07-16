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
void objc_setAssociatedObject(id object, const void * key,id value, objc_AssociationPolicy policy)
```
#### 获得关联对象
```Objective-C
id objc_getAssociatedObject(id object, const void * key)
```
#### 移除所有的关联对象
```Objective-C
void objc_removeAssociatedObjects(id object)
```
#### 常用用法：
```Objective-C
static void *MyKey = &MyKey;
objc_setAssociatedObject(obj, MyKey, value, OBJC_ASSOCIATION_RETAIN_NONATOMIC)
objc_getAssociatedObject(obj, MyKey)
```
**使用get方法的@selecor作为key**
```Objective-C
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

# Author
jiangys, jys509@126.com

# License
iOSInterviewQuestions is available under the MIT license. See the LICENSE file for more info.
