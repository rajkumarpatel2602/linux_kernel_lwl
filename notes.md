# Introduction

- `$uname -r`
- Moduels are prenet in /lib/modules/'uname -r' 
- `$cd /lib/modules/'uname -r'`  and then  `$find . -name "*.ko"`
- To use modules check config `ls /boot/configs'uname -r' `
- in-tree module and out-of-tree module

# Basic commands
- lsmod list already loaded modules from /proc/module or /sys/module. Last loaded will be printed first. name, size, dependent or user module
- modinfo print information about module. also talks about version, in-tree, author and param info which can be used during loading.
- `insmod` insert module
- `rmmod` remove module

# HelloWorld kernel module
- Every module has to provide 2 functions. 'start' (Initialization) and 'end'(CleanUp) function.
- To specify start and end functions, kernel macros are used. e.g. module_init() and module_exit()
- Speicify license. MODULE_LECENSE()
- Header is needed for macros inclusion, "#include linux/module.h" for init and exit macros, and for printk logging, "#include linux/kernel.h"

- hello.c file
```
#include <linux/kernel.h>
#include <linux/module.h>

MODULE_LICENSE("GPL");

static init test_hello_init(void)
{
  printk(KERN_INFO, "%s in INIT", __func__);
  return 0;
}

static void test_hello_exit(void)
{
  printk(KERN_INFO, "%s in INIT", __func__);
}

module_init(test_hello_init);
module_exit(test_hello_exit);
```

# Building the module
- '$make modules' is the command
- `make` command must get called with -C option to tell which kernel Makefile to use.
- This kernel makefile should get passed another makefile, which is module makefile, and have only module `obj-m := hello.o` in it to notify .ko name. obj-y in case of in-tree.
- `$make -C /lib/module/'uname -r'/build M=${pwd} modules`
- $ls will show hello.ko
- $modinfo ./hello.ko
- To clean module `$make -C /lib/module/'uname -r'/build M=${pwd} clean` (All generated files got removed)
- All these is native compilation.
- For cross compilation modify default kernel macro values for ARCH and CROSS_COMPILE
- `$make ARCH=arm CROSS_COMPILE=arm-buildroot-linux-uclibceabi- -C build_makefile_for_arm M=${PWD} modules`
- make sure to omit gcc in CROSS_COMPILE option. -C makefile is using different file as /build/modules are build for x86

# Loading the module
- $sudo insmod hello.ko (only root can run this. check if root or not by whoami)
- `$lsmod | less` (last loaded will be listed first)
- `$ls /sys/module/hello/` is also created
- dmesg to see start printk messages
- `$sudo rmmod hello` (now .ko not required)
- `dmesg` to see cleanup printk messages
- observation in dmesg. out-of-tree will taints kernel message.

# printf vs printk
- no log levles in printf. printf is in standard C library.
- printk is kernel levle function
- printk always called as one more arg. printk(KERN_log_levle"message");
- dmesg prints kernel buffer. printk dump messages to kernel buffer, which can be checked by dmesg.
- printf writes to terminal or stdout.

# Simplified Makefile

- make (default target 'all' will get called)
- make clean

## Makefile
```
obj-m := hello.o

all:
  $make -C /lib/module/'uname -r'/build M=${pwd} modules
clean:
  $make -C /lib/module/$(shell uname -r)/build M=${pwd} clean
```
# insmod callstack
- insmod -> init_module(user space) -> sys_init_module(system call) -> do verify user -> load_module call -> verify elf and alloc memory for module code(elf code) -> return offest to kernel -> add module info to doubly list -> module_init() is called and we can see dmesg print.

# What happen if we return -1 from module_init()
- ![image](https://github.com/user-attachments/assets/dfbb9cfb-2910-437b-9574-ca5ff0410a1a)

# Change module name
- edit Makefile
```
obj-m := linux.o
linux-objs := hello.o // these are required object file to create obj-m
```
![image](https://github.com/user-attachments/assets/744a576e-d923-459c-a54e-2d946a9263be)

# Compiling module with multiple files
- Just update linux-objs := hello.o func.o // and so on
![image](https://github.com/user-attachments/assets/0956dd29-5b38-47aa-be73-6102648808cf)
![image](https://github.com/user-attachments/assets/cf482912-1047-4694-a113-bef22dd203f4)

# Compiling 2 modules with one makefile
- use obj-m := 1.o and then obj-m := 2.o and keep on using +=
- user obj-m  := 1.o 2.o 3.o
- `$sudo insmod 1.ko`
- `$sudo insmod 2.ko`
- `$sudo rmmod 1 2`
![image](https://github.com/user-attachments/assets/82be947a-69b9-4f86-a95b-ea20dec2ce72)

# dmesg in deep
- When kernel bootsup, before mounting of file system, bootings logs needs preservation. For this, kernel users a ring buffer.
- This ringbuffer takes all the printk dump in it, and can be viewed by dmesg.
- Once sysfs is mounted, based on distribution a file will hold printk dump along with ringbuffer.
- dmesg is used to control contents of ring buffer or to print the ring buffer. Default is to print contents on console.
- collected logs are getting stored in /var/logs/dmesg
- `$ps -ef | grep syslog` // can see syslog daemon running.
- syslogd is periodically reading kernel buffer and writing to some file, decided by distribution. e.g. /var/log/kern.log
- `$dmesg -c` // clear after dumping logs on console the ring buffer.
- `$dmesg -C` // simply clear buffer
- `$dmesg -t` // won't print time stamp
- `$dmesg -T` // convert epoch timestamp in human readable format
- `$dmesg -l info, err` // will print only info and err messgages
- `$dmesg -x` // will print loging level information
- `$strace dmesg` // shows from perticular file message is being read and dumped on console by default.
