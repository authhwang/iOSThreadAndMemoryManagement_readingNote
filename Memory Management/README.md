# 内存管理

1. 内存管理最重要的是这4点

* 自己生成的对象，自己所持有
* 非自己生成的对象，自己也能持有
* 不再需要自己持有的对象时释放
* 非自己持有的对象无法释放

这四点无论是arc还是mrc都是非常受用的！！

2. 只有**alloc/new/copy/mutabaleCopy**的才是自己生成并持有 别的都不是 包括init方法 它只是对创建的对象初始化并返回而已
3. 不是**以上面4大金刚开头**的构造方法都是非自己生成的 例如[NSMutableArray array] 所以对于调用方来说你不是主动持有的 在mrc下除非你主动retain 而且！对于这种构造方法 它的内部实现其实是生成该对象后会调用autorelease 把这个对象让autoreleasepool所管理 
4. **autoreleasepool**是会根据**runloop**的休眠或者**runloop**的挂起所释放的 每当有对象调用autorelease时 就会把该对象加入pool中 并持有 当pool需要释放时 就会对pool中的所有对象调用release方法
5. 获取引用计数方法的内部实现是 引用数 + 1 原因是在release内部判断时 是先判断引用数是否为零 是就dealloc 
6. 对于苹果对内存管理方法的实现上 虽然是说当时用引用计数散列表 可是现在是判断用tagged pointer、isa指针获取、引用计数散列表这三种方法其一来获取
7. 引用计数表的好处是各记录中存有内存块地址 可从各个记录追溯到各对象的内存块
8. 在arc管理下 有4个修饰符 

* __strong
* __weak
* __unsafe\_unretained
* __autoreleaseing

除了__unsafe\_unretained外 所有附有修饰符的变量初始化为nil

### __strong

1. 是默认的所有权修饰符 表示自动的持有并在超过变量作用域后自动释放
2. 如果用在非自己生成的对象上 也会持有并在其作用域后释放
3. 当变量被赋值了别的对象 会释放该变量对之前对象的持有

因此__strong基本上可以满足内存管理的思考方法

在arc实现上

针对**4大金刚**的方法 编译器会编译成这样

```objective-c
id obj = objc_msgSend(NSObject, @selcetor(alloc));
objc_msgSend(obj, @selector(init));
objc_release(obj);
```

在变量的作用域结束时会调用objc_release释放对象

针对非4大金刚的方法 编译器会编译成这样 (例如NSMutableArray)

```objective-c
id obj = objc_msgSend(NSMutableArray, @selector(array));
objc_retainAutoreleasedReturnValue(obj);
objc_release(obj);
```

对应array的方法的内部实现会编译这样

```objective-c
id obj = objc_msgSend(NSMutableArray, @selector(alloc));
objc_msgSend(obj, @selector(init));
return objc_autoreleaseReturnValue(obj);
```

objc_retainAutoreleasedReturnValue 主要是针对从autoreleasepool中获取到的对象进行retain

objc_autoreleaseRetrunValue会检查使用该函数的方法或函数调用方的执行命令列表 如果方法或函数的调用方在调用方法后紧接着调用objc_retainAutoreleasedReturnValue  是将对象加到pool中并返回

objc_autoreleaseRetrunValue会检查使用该函数的方法或函数调用方的执行命令列表 如果方法或函数的调用方在调用方法后紧接着调用objc_retainAutoreleasedReturnValue 则不把对象加到pool中 而是加到线程局部存储中并返回对象  可以减少对pool的性能消耗

### __weak

1. 用于解决循环引用的问题 不会持有任何对象
2. 当赋值的对象被废弃时 会自动设置为nil
3. 当调用附有\_\_weak修饰的变量 就将该对象注册到autoreleasepool中

在arc的实现上

针对第2条 实现的原理是

1. 在赋值给\_\_weak修饰符的变量时 会以对象的地址为键值 以变量的地址的键添加到weak散列表中 当超出变量作用域时 会通过键值快速寻找所有对应的记录 然后将键值设置为nil 并将该记录删除
2. 假如是所赋值的对象被废弃时 会在dealloc方法调用后 会通过键值快速寻找所有对应的记录 然后将键值设置为nil 并将该记录删除 （假如通过引用计数表计数的话 也会删除该记录）

针对第1条 实现的原理是 (在编译器上)

会在将记录生成后 由于发现没有被持有 所以就立刻调用objc_release(obj)

针对第3条 实现的原理是

```objective-c
id obj1;
objc_initWeak(&obj1,obj);
id tmp = objc_loadWeakRetained(&obj1);
objc_autorelease(tmp);
objc_destroyWeak(&obj1);
```

objc_loadWeakRetained()做的是将被\_\_weak修饰的变量所赋值的对象给retain并赋值到tmp中

objc_autorelease则是将tmp中的对象添加到pool中并其管理 

所以当被赋值的\_\_weak变量被很多次调用时 则**该对象也会疯狂加到pool里**

最好的解决方法是：

将\_\_weak修饰的变量赋值给\_\_strong修饰的变量即可避免此类问题

###  __unsafe\_unretained

1. 跟__weak一样作用一样 但它不会当对象被废弃时自动设置为nil 所以容易导致崩溃现象

在arc实现的原理是

就等于直接获取对象后 立刻释放对象

### __autoreleasing

1. 当有需要让对象注册autoreleasepool时 直接显示修饰即可
2. 针对非**4大金刚**开头的 都会自动将对象注册到最近的pool中 所以无需显式修饰
3. 当访问__weak修饰的变量时 会将其注册到autoreleasepool中
4. 使用对象类型指针时 会默认修饰为\_\_autoreleasing 而且该变量修饰符与对象指针的修饰符需要一致 虽然有时候在对于传入对象指针为参数的函数中 可以在创建变量时是无须显式修饰\_\_autoreleasing 因为在编译器编译时会帮你处理


在arc的实现 编译器编译的是这样

针对4大金刚:

```objective-c
@autoreleasepool {
  id __autoreleasing obj = [[NSObject alloc] init];
}

/* 编译器的代码 */
id pool = objc_autoreleasePoolPush();
id obj = objc_msgSend(NSObject, @selector(alloc));
objc_msgSend(obj, @selector(init));
objc_autorelease(obj);
objc_autoreleasePoolPol(pool);
```

针对非4大金刚:

```objective-c
@autoreleasingpool {
  id __autoreleasing obj = [NSMutableArray array];
}

/* 编译器的代码 */
id pool = pbjc_autoreleasePoolPush();
id obj = objc_msgSend(NSMutableArray, @selector(array));
objc_retainAutoreleaseReturnValue(obj);
objc_autorelease(obj);
objc_autoreleasePoolPop(pool);
```

对于非4大金刚来说 在没用\_\_autoreleasing修饰前 就已经是有可能被加入autoreleasepool了(除了进入优化机制下)  修饰后 则将其再被加入pool中 要注意这一点


9. Core Foundation 框架与 Foundation框架之间的转换

* __bridge 的话是直接转换不会增加引用计数
* \_\_bridge\_retained或者CFBridgingRetain函数（内部实现也是用\_\_bridge\_retained）会增加该对象的引用计数
* \_\_bridge_transfer或者CFBridgingRelease函数 （内部实现是用\_\_bridge_transfer）会把对象的持有转换给新的变量 旧的变量则释放

10. property

* assign对应是\_\_unsafe\_unretained
* copy对应是\_\_strong 不过是通过copyWithZone:方法复制值源所生成的对象
* retain对应是\_\_strong
* strong对应时\_\_strong
* unsafe\_unretained对应是\_\_unsafe\_unretained
* weak对应时\_\_weak

11. 数组
    1. 针对静态数组时会根据数组的修饰符对数组中的元素的修饰符也是一样
    2. 针对动态数组 假如是foundation框架的容器 会适当地持有追加的对象并管理这些对象 假如是c语言的动态数组 则需要在创建的数组的时候要用到calloc函数 因为会将分配区域初始化为0 调用malloc函数的话还需调用memset将内存填充为0 不能直接赋值为nil 因为nil会被赋值给随机地址的变量中 当释放数组的时候也不能直接释放 要将每个元素都赋值为nil来释放对象的引用 假如直接释放数组会引起内存泄漏