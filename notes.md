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
![image](https://github.com/user-attachments/assets/ae911577-06ff-4576-983f-1233c472d501)

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
- `$dmesg -w or $dmesg --follow // this will keep printing dmesg whenever there will be addition of logs to buffer.
- Make dmesg with -w option running in background and whatever debugging or testing we'll be doing, we will have dmesg logs.
- `$dmesg -w &` this is the command to make it run in background.
![image](https://github.com/user-attachments/assets/884f0b2c-74dd-4d13-b4ed-e5c69e4ed078)

# What happens when exit function is not present
- Check `linux_src/kernel/module.c`, SYSCALL_DEF* and check delete_module. it checks init_module and exit_module presence. if not send -EBUSY
- `$errno EBUSY`
![image](https://github.com/user-attachments/assets/351e8576-7d75-4983-b354-583b47f99d7c)
- To remove, reboot required.

# What happens when init function is not present
- Compiles and gets loaded without init function
![image](https://github.com/user-attachments/assets/a3538ddd-2313-4830-b417-6f937cebeb1b)
- insmod and rmmod both worked.

# Are init and exit are mandatory?
- Check this smallest kernel module
![image](https://github.com/user-attachments/assets/a187159f-7cf1-4355-9dfe-0d3f9eb0f398)
- Used when, a LKM is having a lot of functional implementation, which are being used by other LKM.

# Cross compilation
![image](https://github.com/user-attachments/assets/e092ff4c-624c-434f-9ca0-c07674a94dfe)
- $file hello.ko // shows the attributes of the file. (https://www.udemy.com/course/learn-linux-kernel-programming/learn/lecture/22139088#reviews)

# .c to .ko using kbuild
![image](https://github.com/user-attachments/assets/34b86444-33a9-4c9f-a3b1-8c5267d1aa5c)
![image](https://github.com/user-attachments/assets/33216913-661a-4255-844b-9eeeba0771ad)
.c -> .o, .mod.order(order of module creation in case of multiple modules), .mod.symvers(contains symbols exported by Module?) .mod.c (contains version info) -> .mod.o (& .o are linked by modpost) -> .ko

# modprobe vs insmod
- Uses standard path (/lib/modules/'uanme -r') and also resolves module dependencies
![image](https://github.com/user-attachments/assets/ba561480-07e8-4339-baef-e7c4b87dad4e)
- Uses `depmod`, which generates modules.dep and map files
- `$cd /lib/modules/'uname -r'` and then `$ls && cat modules.dep`
![image](https://github.com/user-attachments/assets/ec7a4fe5-6468-4e2b-a235-453c0827431a)
- Modules are added in right to left, and while removing, Modules are removed from left to right from .dep file.
- Reload modules.dep by `$depmod -a`
- depmod calculates dependencies of all the  modules present in /lib/modules/$(uname -r) folder, and places the dependency information in /lib/modules/$(uname -r)/modules.dep file

# Typechecking of module_init() and module_exit() functions
![image](https://github.com/user-attachments/assets/63f55d81-02e8-4e8b-a483-c3d8d4625224)
- observe aliasing of passed functions to cleanup_module and init_module.

# Passing args to module
- module_param(name, type, permission) // e.g. module_name(loop_cnt, int, IORUGO) // #include <linux/moduleparam.h>
![image](https://github.com/user-attachments/assets/728a0efd-e0c0-4528-8c67-6218d966844a)
![image](https://github.com/user-attachments/assets/73a008a1-b433-41cd-ab0c-5eed0eb4bcc0) 
![image](https://github.com/user-attachments/assets/b9d6af6c-0d42-43f7-9f9f-7acc9347ba9b)
![image](https://github.com/user-attachments/assets/474402dd-ebab-4691-8626-72d2d5c0a14a)
- in /sys/modules/<module_name>/parameteres/<parameter_name> check params and permissions while loading
![image](https://github.com/user-attachments/assets/dd9a3f4f-3f0f-42fd-a36f-ab87a5dcada6)
- if passed permission as 0, then sysfs file for that parameter won't be created.
- module_param_array(name, type, nump, permission) // e.g. module_name(loop_cnt, int, &args, IORUGO) // #include <linux/moduleparam.h>
- `$sudo insmod argument.ko param_array=1, 2, 3, 4`
- `$sudo insmod argument.ko param_array=1, 2, 3, 4, 5` here, loading will fail.
![image](https://github.com/user-attachments/assets/637383df-77a9-42f7-a0f9-f6cc1ed9f2b3)

#  From where modprobe uses param values?
- `cat /etc/modprobe.conf` this file is read by modprobe for parameters.
- To pass args to built-in modules in kernel, use kernel command line and set by `<module_name>.<parameter_name>=value`
