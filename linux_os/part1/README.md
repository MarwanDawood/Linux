## Run OS in QEMU
`sudo apt-get install qemu`

Create QEMU image and mount it by running the below script\
`./qemu/qemu-run.sh`

## Run OS in VM
* install VirtualBox from Oracle
* install Kali Linux 10GB disk size
* create 1 virtual hard disk in virtualbox and add it to x64 Kali machine
* restart your virtual machine.
* run `lsblk` to see that changes took effect
* change user to super user, `sudo su`
* run `cfdisk /dev/sdb` to create a partition `dos, bootable, primary`
* run `lsblk` to see that changes took effect, **part** type shall appear under **disk** type.

### Prepare filesystem
* format it using `mkfs.ext4 /dev/sdb1`
* `cd /mnt`
* `mkdir myos`
* mount sdb1 as myos `mount /dev/sdb1 myos`
* `cd myos`
* delete the folder lost+found `rm -rf lost+found`
* add the basic linux directories `mkdir -pv ./{bin,sbin,etc,lib,lib64,var,dev,proc,sys,run,tmp,boot}`, p -> parents no error if existing, v -> verbose
* `mknod -m 600 ./dev/console c 5 1`, create a character device version 5.1
* `mknod -m 666 ./dev/null c 3 1`, create a null device

### Prepare boot
* copy the initial RAM disk from current machine, `cp /boot/initrd.img-5.10.0-kali7-amd64 .`
* copy the machine kernel from current machine as a temporary solution, `cp /boot/vmlinuz-5.10.0-kali7-amd64 .`
* `grub-install /dev/sdb --skip-fs-prob --boot-directory=/mnt/myos/boot` to install grub in the boot directory
* create this config file, `vim grub/grub.cfg` and add the following:
```
    set default=0
    set timeout=30
    menuentry "myOs 0.0.1" {
        linux /boot/vmlinuz-5.10.0-kali7-amd64 root=/dev/sda1 ro
        initrd /boot/initrd.img-5.10.0-kali7-amd6
    }
```

### Create startup code
* add the source files into /mnt/myos/src
* add the source files `init.c` and `start.S` in this folder.
* compile the src files using `gcc -nostdlib -ffreestanding -no-pie init.c start.S`
* replace the binary file in sbin/init with the generated one `cp a.out ../sbin/init`
### How ABI work for x64?
```
_syscall:
    movq %rdi, %rax         /* Syscall number -> rax.  */
    movq %rsi, %rdi         /* shift arg1 - arg5.  */
    movq %rdx, %rsi
    movq %rcx, %rdx
    movq %r8, %r10
    movq %r9, %r8
    movq 8(%rsp),%r9        /* arg6 is on the stack.  */
    syscall                 /* Do the system call.  */
    cmpq $-4095, %rax       /* Check %rax for error.  */
    jae SYSCALL_ERROR_LABEL /* Jump to error handler if error.  */
    ret                     /* Return to caller.  */
```
#### 4.4.1 Calling a Function
To call a function, the program should place the first six integer or pointer parameters in the
6 registers <200b>%rdi<200b>, <200b>%rsi<200b>, <200b>%rdx<200b>, %rcx<200b>, <200b>%r8<200b>, and <200b>%r9<200b>; subsequent parameters (or parameters larger than 64 bits) should be pushed onto the stack, with the first argument topmost.

#### 4.3 Register Usage
By convention, <200b>%rax<200b> is used to store a function’s return value, if it exists and is no more than 64 bits long. Additionally,<200b> %rdi<200b>, %rsi<200b>, %rdx , %rcx, %r8<200b>, and %r9<200b> are used to pass the first 6 integer or pointer parameters to called functions. Additional parameters (or large parameters such as structs passed by value) are passed on the stack.

#### why %r10 is used instead of %rcx?
https://stackoverflow.com/questions/32253144/why-is-rcx-not-used-for-passing-parameters-to-system-calls-being-replaced-with
wpZhsZ4oCdVe&XwpZhsZ4oCdVe&X
