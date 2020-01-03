# linux内核中有关GNU-C的学习笔记
##一、简介
在linux内核源码中，常出现一些并不非常符合c语言语法的代码，但他又确实写在了c文件中。类似下面这种初始化结构体
<h1>static const struct file_operations ab3100_otp_operations = {
.open        = ab3100_otp_open,
.read        = seq_read,
.llseek        = seq_lseek,
.release    = single_release,
};</h1>
查阅资料后发现，这些语法是GNU C扩展的c语言语法。于是用这篇文来记录在学习linux内核中有关gnu c的一些笔记。
##二、指定初始化
