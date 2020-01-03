# linux内核中有关GNU-C的学习笔记

## 一、简介
在linux内核源码中，常出现一些并不非常符合c语言语法的代码，但他又确实写在了c文件中。类似下面这种初始化结构体

```
static const struct file_operations ab3100_otp_operations = {
.open        = ab3100_otp_open,
.read        = seq_read,
.llseek        = seq_lseek,
.release    = single_release,
}; 
```

查阅资料后发现，这些语法是GNU C扩展的c语言语法。于是用这篇文来记录在学习linux内核中有关gnu c的一些笔记。

## 二、指定初始化
下面对比几种语法间的差异
# 在数组中
1.标准c

```int c[5]={1,2,3,4};```

固定顺序赋值，没有被赋值的c[4]自动赋值0
2.c99

```int c[5]={[1]=2,[3]=4};```

可指定某个数组元素赋值，更加自由
3.GNU C

```int c[5]={[1]=2,[3..4]=4}```

GNU C支持c99，而且还有GNU 支持使用..表示范围扩展例如[3..4]代表3和4，[1..100]代表1到100
# 在结构体中

```
struct goods{
    char name[10];
    int price;
    int number;
};
```

1.标准c

```
struct goods goods1=
{
  "apple",
  5,
  100
}
```

类似数组，标准c在结构体的赋值时也是需要固定顺序赋值
2.GNU C

```
struct goods goods2=
{
  .name = "orange",
  .price = 10
}
```
在GNU C中，可以通过结构域名来为特定的成员赋值
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
