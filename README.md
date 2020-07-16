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
图片1，图片2
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

图片2

### block 里面的变量为什么要用copy
### weak原理
### 离屏渲染怎么产生，怎么避免
### tableView性能优化

# Author
jiangys, jys509@126.com

# License
iOSInterviewQuestions is available under the MIT license. See the LICENSE file for more info.
