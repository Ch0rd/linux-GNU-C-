# linux内核中有关GNU-C的学习笔记

## 一、简介
在linux内核源码中，常出现一些并不非常符合c语言语法的代码，但他又确实写在了c文件中。
类似下面这种初始化结构体

```
static const struct file_operations ab3100_otp_operations = {
.open        = ab3100_otp_open,
.read        = seq_read,
.llseek        = seq_lseek,
.release    = single_release,
}; 
```

这些语法是GNU C扩展的c语言语法。

## 二、指定初始化
对比几种语法间的差异
### 在数组中
1.标准c

```int c[5]={1,2,3,4};```

固定顺序赋值，没有被赋值的c[4]自动赋值0
2.c99

```int c[5]={[1]=2,[3]=4};```

可指定某个数组元素赋值，更加自由
3.GNU C

```int c[5]={[1]=2,[3..4]=4}```

GNU C支持c99，而且还有GNU 支持使用..表示范围扩展例如[3..4]代表3和4，[1..100]代表1到100
### 在结构体中

```
struct goods{
    char name[10];
    int price;
    int number;
};
```

### 1.标准c

```
struct goods goods1=
{
  "apple",
  5,
  100
}
```

类似数组，标准c在结构体的赋值时也是需要固定顺序赋值。

### 2.GNU C

```
struct goods goods2=
{
  .name = "orange",
  .price = 10
}
```
在GNU C中，可以通过结构域名来为特定的成员赋值。

于linux内核中，结构体赋值常用于内核驱动的初始化中

```
static const struct file_operations ab3100_otp_operations = {
.open        = ab3100_otp_open,
.read        = seq_read,
.llseek        = seq_lseek,
.release    = single_release,
};
```

常使用 file_operations 这个结构体变量来注册开发的驱动，然后以回调的方式来执行我们驱动实现的相关功能。结构体 file_operations 在 Linux 内核中的定义如下：

```
struct file_operations {
        struct module *owner;
        loff_t (*llseek) (struct file *, loff_t, int);
        ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
        ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
        ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);
        ssize_t (*write_iter) (struct kiocb *, struct iov_iter *);
        int (*iterate) (struct file *, struct dir_context *);
        unsigned int (*poll) (struct file *, struct poll_table_struct *);
        long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
        long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
        int (*mmap) (struct file *, struct vm_area_struct *);
        int (*open) (struct inode *, struct file *);
        int (*flush) (struct file *, fl_owner_t id);
        int (*release) (struct inode *, struct file *);
        int (*fsync) (struct file *, loff_t, loff_t, int datasync);
        int (*aio_fsync) (struct kiocb *, int datasync);
        int (*fasync) (int, struct file *, int);
        int (*lock) (struct file *, int, struct file_lock *);
        ssize_t (*sendpage) (struct file *, struct page *, int, size_t, loff_t *, int);
        unsigned long (*get_unmapped_area)(struct file *,
               unsigned long, unsigned long, unsigned long, unsigned long);
        int (*check_flags)(int);
        int (*flock) (struct file *, int, struct file_lock *);
        ssize_t (*splice_write)(struct pipe_inode_info *, 
            struct file *, loff_t *, size_t, unsigned int);
        ssize_t (*splice_read)(struct file *, loff_t *, 
            struct pipe_inode_info *, size_t, unsigned int);
        int (*setlease)(struct file *, long, struct file_lock **, void **);
        long (*fallocate)(struct file *file, int mode, loff_t offset,
                  loff_t len);
        void (*show_fdinfo)(struct seq_file *m, struct file *f);
        #ifndef CONFIG_MMU
        unsigned (*mmap_capabilities)(struct file *);
        #endif
 };
```  

 结构体 file_operations 里面定义了很多结构体成员，而在这个驱动中，我们只初始化了部分成员变量，通过访问结构体的成员来指定初始化，非常方便。

## 三、宏构造拓展
### 语句表达式
GNU C 对 C 标准作了扩展，允许在一个表达式里内嵌语句，允许在表达式内部使用局部变量、for 循环和 goto 跳转语句。
语句表达式的格式如下：
({ 表达式1; 表达式2; 表达式3; })
语句表达式的值为内嵌语句中最后一个表达式的值。例如

```
sum = 
({
    int s = 0;
    for( int i = 0; i < 10; i++)
    s = s + i;
    s;
});
```

sum的值为1到10的求和。
### 在宏定义中使用
题目：定义一个宏，比较两个值得大小

```
#define max(a,b) ((a)>(b)?(a):(b))
```
bug:当出现max(i++,j++)时，i和j的值在宏展开时会自增两次，造成错误。
通过加入语句表达式，增加两个临时变量储存值再比较。

```
#define max(type,a,b)({    \
    type _a = a;           \
    type _b = b;           \
    _a > _b ? _a : _b      \
})
```

添加参数type指定临时变量类型，可以比较任意类型的数据。

## 四、关键字typeof
GNU C扩展了一个关键字typeof，用来获取一个变量或表达式的类型。注意，typeof还没有被写入C标准。
使用 typeof，可以获取一个变量或表达式的类型。
```
int i ;
typeof(i) j = 20;//变量i的类型为int，所以typeof(i)就等于int

typeof(int *) a;//typeof(int *)就等于int *

int f();
typeof(f()) k;//表达式f()的类型为int，所以typeof(f())就等于int
```

### typeof的高级用法
```
typeof (int *) y;  // 把 y 定义为指向 int 类型的指针，相当于int *y;
typeof (int)  *y;  //定义一个执行 int 类型的指针变量 y
typeof (*x) y;     //定义一个指针 x 所指向类型 的指针变量y
typeof (int) y[4]; //相当于定义一个：int y[4]
typeof (*x) y[4];  //把 y 定义为指针 x 指向的数据类型的数组
typeof (typeof (char *)[4]) y;//相当于定义字符指针数组：char *y[4];
typeof(int x[4]) y;//相当于定义：int y[4]
```

### typeof在linux内核中的应用
```
#define min(x, y) ({                \
    typeof(x) _min1 = (x);          \
    typeof(y) _min2 = (y);          \
    (void) (&_min1 == &_min2);      \
    _min1 < _min2 ? _min1 : _min2; })

#define max(x, y) ({                \
    typeof(x) _max1 = (x);          \
    typeof(y) _max2 = (y);          \
    (void) (&_max1 == &_max2);      \
    _max1 > _max2 ? _max1 : _max2; })
```

在 min\max 宏定义中，使用 typeof 直接获取参数类型。
其中有一行代码
```
   (void) (&_max1 == &_max2);
```

用来检测宏的两个参数 x 和 y 的数据类型是否相同。如果不相同，编译器会给一个警告信息。
```
warning：comparison of distinct pointer types lacks a cast
```

语句 &_max1 == &_max2 用来判断两个变量 _max1 和 _max2的地址是否相等，即比较两个指针是否相等。当两个变量类型不相同时，指针类型也不同。比如一个 int 型变量，一个 char 变量，对应的指针类型，分别为 char * 和 int *，而两个指针比较，它们必须是同种类型的指针，否则编译器会有警告信息，提示两种数据类型不一致。

而由于比较结果的值并不会用到，编译器也会报警告信息，通过加上(void)将此种警告信息去除。

## 五、container of 宏
```
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)
#define  container_of(ptr, type, member) ({    \
     const typeof( ((type *)0)->member ) *__mptr = (ptr); \
     (type *)( (char *)__mptr - offsetof(type,member) );})
```

主要作用：根据结构体某一成员的地址，获取这个结构体的首地址。

这个宏有三个参数，它们分别是：
type：结构体类型
member：结构体内的成员
ptr：结构体内成员member的地址

### 使用示例
```
struct team
{
    int a;
    int b;
    int c;
};
int main(void)
{
    struct team t_one;
    struct team, *p;
    p = container_of( &t_one.c, struct team, c);
    return 0;
}
```

Linux 内核驱动中，为了抽象，对数据结构体进行了多次封装，往往一个结构体里面嵌套多层结构体。也就是说，内核驱动中不同层次的子系统或模块，使用的是不同封装程度的结构体。这是面向对象思想。分层、抽象、封装，让程序兼容性更好，适配更多的设备，但同时也增加了代码的复杂度。
我们在内核中，经常会遇到这种情况：我们传给某个函数的参数是某个结构体的成员变量，然后在这个函数中，可能还会用到此结构体的其它成员变量。通过container_of，我们可以首先找到结构体的首地址，然后再通过结构体的成员访问就可以访问其它成员变量。

###container_of实现

计算结构体成员在结构体内的偏移：
```
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)
```

将0强制转换为一个指向 TYPE 的结构体常量指针，然后通过这个常量指针访问成员，获取成员 MEMBER 的地址，其大小在数值上等于 MEMBER 在结构体 TYPE 中的偏移。
size_t:unsigned int 无符号整型
&((TYPE *)0)->MEMBER:MEMBER在TYPE中的偏移

结构体的成员数据类型可以是任意数据类型。为了让宏兼容各种数据类型，定义了一个临时指针变量__mptr，该变量用来存储结构体成员 MEMBER 的地址，即存储 ptr 的值。

```
const typeof( ((type *)0)->member ) *__mptr = (ptr); \
```

宏的参数 ptr 代表的是一个结构体成员变量 MEMBER 的地址，所以 ptr 的类型是一个指向 MEMBER 数据类型的指针。当使用临时指针变量__mptr来存储 ptr 的值时，必须确保__mptr 的指针类型是一个指向 MEMBER 类型的指针变量。表达式使用 typeof 关键字，用来获取结构体成员 member 的数据类型。

```
(type *)( (char *)__mptr - offsetof(type,member) );
```

结构体成员 member 的地址，减去在结构体 type 中的偏移，结果就是结构体 type 的首地址。结果也是整个语句表达式的值，container_of 最后会返回这个地址值。
在语句表达式的最后，因为返回的是结构体的首地址，所以数据类型还必须强制转换为 TYPE* ，即返回一个指向 TYPE 结构体类型的指针。

