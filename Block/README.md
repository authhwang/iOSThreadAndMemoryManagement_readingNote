# Block

block就是一个带有局部变量值的匿名函数

1. block表达式跟平时的函数表达式几乎一样 ^void (int event) {} 就是没了函数名和多了个^
2. 表达式可以省略返回值类型和参数列表(若无参数的情况下)
3. block类型跟平时的函数指针的定义几乎一样 int (^blk) (int) 就是多了个^
4. int (^func () (int)) 看到这个的时候 它所指是返回func函数返回值为block

## block的实现

1. block在编译后其实是一个结构体实例 也是一个object-c对象（拥有isa指针）

   ```c
   struct __block_impl {
     void *isa;
     int Flags;
     int Reserved;
     void *FuncPtr;
   }

   struct __main_bolck_impl_0 {
     struct __block_impl impl;
     struct __main_block_desc_0* Desc;
     
     __main_blockimpl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
       impl.isa = &_NSConcreteStackBlock;
       impl.Flags = flags;
       imp.FuncPtr = fp;
       Desc = desc;
     }
   };

   static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
     printf("Blcok\n");
   }

   static struct __main_block_desc_0 {
     unsigned long reserved;
     unsigned long Block_size;
   }

   __main_block_desc_0_DATA = {
     0,
     sizeof(struct __main_block_impl_0)
   };

   int main() {
       void (*blk)(void) = (void (*)(void))&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA);
     
     ((void (*)(struct __block_impl *))((struct __block_impl *)blk)->FuncPtr)((struct __block_impl *)blk)
     
     return 0;
   }
   ```

   ​

2. 针对它能截获局部变量值（非对象）的原理是：

block的结构体是会在block表达式使用的变量值的来作为结构体的成员变量 因此它能截获变量值 不会因为需要使用的变量值在block表达式创建后改变而改变 而且在block里面的表达式时 编译器会取出block结构体的该成员变量值 重新赋予给表达式里面所用的变量 所以看上去就是这么的自然～

```c
staruct _main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  const char *fmt;	//在block中使用的变量值
  int val;			//在block中使用的变量值
}
```

可是c语言数组类型不能在block中使用 因为在c语言数组类型不能赋值给c语言数组类型

3. \_\_block说明符的原理 （修饰非对象时）

被\_\_block修饰的变量 都会变成新的结构体实例 （也是一个object-c对象）并把当时的变量值给保存在该结构体的成员变量val中 然后该结构体实例也会被block结构体实例作为成员变量所**截获** 所以当需要在block表达式中进行变量赋值时 其实改变的就是\_\_block的结构体实例的成员变量val

```c
struct __Block_byref_val_0 {
  void *__isa;
  __Block_byref_val_0 *__forwarding;
  int __flags;
  int __size;
  int val;
}

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0 *Desc;
  __Block_byref_val_0 *val;	//block结构体也会拥有__Block_byref_val_0的对象
  ...
}
```



4. block的存储域

* _NSConcreteStackBlock  — 栈
* _NSConcreteGlobalBlock — 程序的数据区域
* _NSConcreteMallocBlock — 堆

这三个其实是都是类对象 在block结构体实例化的时候赋值在isa指针上的 

假如block是在用在全局变量中或者block表达式中没有用到任何局部变量时 则会将isa指针赋值_NSConcreteGlobalBlock

假如block是用在函数的返回值、主动调用copy方法、作为GCD或者平时cocoa框架方法作为方法参数、将block赋值给赋有\_\_strong修饰符id类型的类或Block类型成员变量时 则会在方法内部调用copy时 则会将isa指针赋值_NSConcreteMallocBlock

那其余情况下 isa指针赋值_NSConcreteStackBlock 

针对block存储在堆时：

1. 假如block作为函数的返回值时 会将block从栈复制到堆中 然后会将堆中的block注册到autoreleasepool中 让pool管理
2. 其他情况下在栈的block复制都只是将堆中的block地址赋给变量
3. 假如copy了在堆中的block时 只会增加引用计数
4. 也会将\_\_block变量从栈复制到堆中 并被在堆中的block结构体实例所**持有** 假如有多个block表达式引用 则增加\_\_block的引用计数 (因为\_\_block结构体也是object-c对象)
5. 当\_\_block结构体实例被复制到堆时 在栈上的\_\_block结构体实例的成员变量forwarding会在复制的时候把本来指向栈上的\_\_blcok结构体实例改为指向堆上的\_\_block结构体实例 因此在复制后 无论对堆上的还是栈上的\_\_block变量进行赋值 改变的都是堆上的值

---

5. 针对截获对象使该对象超越其作用域的原理：

在**不调用copy**的时候 假如在栈上的block表达式中使用变量时 变量可以被block所截获 可是当变量的作用域销毁时 则由于没有强引用的作用下所废弃 无法对对象进行调用

因为c语言结构体是不能含有赋有\_\_strong修饰符的变量 因为无法判断何时进行初始化和废弃操作 不能管理内存

可是**调用copy后**由于object-c的运行库能准确把握何时栈复制到堆以及堆中的block被废弃 因此在复制时通过调用block结构体的方法\_\_main\_block\_copy对对象进行retain操作并赋值在block的结构体成员变量中 在block被废弃时调用block结构体的方法\_\_main\_block\_dispose对变量进行release操作释放结构体成员变量

6. 针对修饰\_\_block修饰的对象使该对象超越其作用域的原理：

原理基本上跟block对变量的持有是一样 不过触发点和持有的地方不同 是要在\_\_block变量被复制到堆上时和在堆上的\_\_block变量被销毁时 而且调用的方法和赋有\_\_strong修饰的成员变量都是在\_\_block结构体中

7. 在arc无效时 \_\_block说明符的作用是用于避免block中的循环引用 所以当block从栈复制到堆时 被\_\_block修饰的的对象类型的变量不会被retain 要注意这一个不同点