# iOSInterviewQuestions
生成索引目录：https://github.com/ekalinin/github-markdown-toc  下载下来，打开gh-md-toc所有的目录，输入./gh-md-toc /Users/yongsheng/VSMVVM/README.md即可

# 索引
  * [KVO的原理](#kvo的原理)
      * [block 里面的变量为什么要用copy](#block-里面的变量为什么要用copy)
      * [weak原理](#weak原理)
      * [离屏渲染怎么产生，怎么避免](#离屏渲染怎么产生怎么避免)
      * [tableView性能优化](#tableview性能优化)
# 题解
## 一个NSObject对象占用多少内存？
系统分配了16个字节给NSObject对象（通过malloc_size函数获得）  
但NSObject对象内部只使用了8个字节的空间（64bit环境下，可以通过class_getInstanceSize函数获得）
在源代码中有对齐逻辑，如果字节小于8，会自动补齐到

## 对象的isa指针指向哪里？
- instance对象的isa指向class对象
- class对象的isa指向meta-class对象
- meta-class对象的isa指向基类的meta-class对象

## OC的类信息存放在哪里？
- 对象方法、属性、成员变量、协议信息，存放在class对象中
- 类方法，存放在meta-class对象中
- 成员变量的具体值，存放在instance对象



## KVO的原理
## block 里面的变量为什么要用copy
## weak原理
## 离屏渲染怎么产生，怎么避免
## tableView性能优化

# Author
jiangys, jys509@126.com

# License
iOSInterviewQuestions is available under the MIT license. See the LICENSE file for more info.
