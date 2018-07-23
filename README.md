# Monitor_Zombie_in_iOS
iOS App自动监控Zombie对象方案

>iOS开发过程或者线上版本经常有Crash崩溃在objc_msgSend、objc_retain、objc_release等方法，这些都是典型等Zombie问题，在开发过程可以使用Instruments工具定位，但对于线上问题或者很难复的问题Instruments就很难定位了，如果能主动捕捉Zombie对象，并且Trace Zombie对象信息和释放栈，那就很容易分析问题了。本文介绍一种主动监控Zombie对象方案，方案已经上线验证一段时间了，并且已经在github上开源了。

## 为什么使用ARC还会有Zombie问题？
可能很多同学觉得使用ARC和weak属性后就不会有Zombie问题了，但App上线后还是会发现很多Zombie问题，主要是因为：
- 还有很多地方使用assign
由于历史原因，系统库还有地方使用assign，最典型的就是**iOS8下UITableView delegate和dataSource**，相信大部分iOS开发都遇到过这个问题导致的Crash，还有就是自己都代码也可能使用assign。
- 线程安全问题
虽然ARC下可以不用考虑对象释放问题，但如果不是线程安全的话，就算使用weak还是可能导致Zombie问题。
## 如何分析Zombie问题？
大多数Zombie问题都是Crash在objc_msgSend方法，一般都是消息转发的时候发生EXC_BAD_ACCESS错误，但从堆栈上无法看出是什么对象、什么方法出了问题，并且很多时候也不是Crash在我们自己但代码附近，这种情况就更难分析了。
我们知道调用**objc_msgSend方法的时候前面几个参数是通过x0~x7寄存器传递的，其中x0是self指针，x1是SEL**，如果能知道Crash在哪个SEL，结合代码有可能进一步分析出哪里Crash了。SEL是指向C string的指针，并且C string 保存在__TEXT段，所以结合dSYM和x1寄存器，正常情况下是可以解析出SEL的，具体解析方法可以参考[「So you crashed in objc_msgSend()」](http://sealiesoftware.com/blog/archive/2008/09/22/objc_explain_So_you_crashed_in_objc_msgSend.html)。
如果SEL是很常见的方法，即使知道SEL还是很分析出问题，这个时候如果能知道Zombie对象类名，对象释放栈的话就能很容易分析出问题了，但仅通过Crash log文件是无法获取这些信息的，这也是大多数Zombie问题很难定位的原因。
其实除了Crash，Zombie也可能只是导致逻辑错误，这个时候就更难定位问题了，因为往往问题现场离对象释放的地方很远。
## 自动监控Zombie对象方案
如果能主动监测Zombie，在第一次使用Zombie对象的时候就发现问题，可以进行上报，也可以主动触发Crash，同时在释放对象的时候保存对象信息，可以知道到底是哪个对象出现了问题，这样可以极大的提高Zombie问题的发现率和解决率。
要在线上版本实时监控，监控组件必须对App性能影响很小，在设计对时候从下面点考虑：
- Trace Zombie对象类名、selector、释放栈信息
- 内存可控
- 监控策略可控
- 对cpu影响要很小
#### 如何监控
要监控Zombie对象，就必须监控访问已经释放对对象，所以可以从下面几个点思考：
1. 如何监控对象释放
Objective-C上可以方便的使用Method swizzling hook dealloc方法监控对象的释放，Method swizzling只能监控Objective-C对象的释放；也可以在更底层hook free，使用Fishhook可以很方便的hook free方法，hook free可以监控所以对象的释放。
2. 如何监控访问已经释放的对象
可以使用Objective-C Runtime消息转发机制监控Objective-C对象访问，要监控C/C++对象访问复杂些，一种方法是对象释放后使用vm_protect设置虚拟内存为不可读写，不过vm_protect只能以内存页为单位进行设置。
#### 最终方案
因为iOS App上大部分自定义对象都是Objective-C对象，所以最终使用Method swizzling hook dealloc方法监控对象的释放，并且使用Runtime消息转发机制监控Objective-C对象访问，主要过程如下：
1. hook dealloc方法，dealloc时只析构对象，不释放内存，更换isa指针指向ZombieHandler Class
2. 延迟释放对象
3. ZombieHandler Class拦截消息，从而监控使Zombie对象
方案模块结构如下图：
![Zombie监控模块结构图](https://github.com/AlexTing0/Monitor_Zombie_in_iOS/raw/master/images/zombie.jpg)
#### 内存优化
一开始对象释放栈保存完整的栈，并且保存string类型，类似下面
> "1   libdispatch.dylib    0x0000000021809823 0x21807000 + 10275"
        "2   libdispatch.dylib    0x0000000021809823 0x21807000 + 10275"

后来改成只保存函数地址，并且arm64下每个地址只用40bit，iOS64位系统下每个地址其实只用了36位，使用40位方便操作，上报的时候也只上报函数地址，这样可以极大程度的减小组件占用的内存，优化后栈类型下面：
>dealloc stack:{
tid:1027
stack:[0x0000000100047534,0x000000010004b2e4,0x00000001000498b0,0x000000018e9bdf9c,0x000000018e9bdb78,0x000000018e9c43f8,0x000000018e9c1894,0x000000018ea332fc,]
}

其实栈可以直接保存在延迟释放对象的内存上面，这样可以进一步优化内存使用。
#### 开源
监控组件已经开源，并且提供符号化脚本，使用也很简单，只需要几行调用就可以：
```
    //setup DDZombieMonitor
    void (^zombieHandle)(NSString *className, void *obj, NSString *selectorName, NSString *deallocStack, NSString *zombieStack) = ^(NSString *className, void *obj, NSString *selectorName, NSString *deallocStack, NSString *zombieStack) {
        NSString *zombeiInfo = [NSString stringWithFormat:@"ZombieInfo = \"detect zombie class:%@ obj:%p sel:%@\ndealloc stack:%@\nzombie stack:%@\"", className, obj, selectorName, deallocStack, zombieStack];
        NSLog(@"%@", zombeiInfo);
        
        NSString *binaryImages = [NSString stringWithFormat:@"BinaryImages = \"%@\"", [self binaryImages]];
        NSLog(@"%@", binaryImages);
    };
    [DDZombieMonitor sharedInstance].handle = zombieHandle;
    [[DDZombieMonitor sharedInstance] startMonitor];
```
**组件支持下面特性：**
- 主动监控Zombie问题，并且提供Zombie对象类名、selector、释放栈信息
- 支持不同监控策略，包括App内自定义类、白名单、黑名单、所有对象
- 支持设置最大占用内存
- 组件在收到内存告警或超过最大内存时，通过FIFO算法释放部分对象
#### 性能影响和稳定性
 组件上线一两个版本了，目前还没发现Crash
 cpu影响：打开Zombie检测前后相差0.2%左右，影响很小
 内存影响：Zombie组件内存开关为10M的时候，实际内存增加11M左右，10M只计算了延迟释放对象和对象释放栈，组件本身占用内存没计算在内，符合预期


具体源码请查看[「DDZombieMonitor」](https://github.com/AlexTing0/DDZombieMonitor) 
## 参考
*[So you crashed in objc_msgSend()](http://sealiesoftware.com/blog/archive/2008/09/22/objc_explain_So_you_crashed_in_objc_msgSend.html)*
