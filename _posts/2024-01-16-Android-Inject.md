# Android Inject

## 原理
attach到目标进程->保存寄存器状态->获取目标进程的mmap,dlopen,dlclose等函数地址->远程调用mmap函数在目标进程中申请内存用来保存参数信息->向目标进程内存空间写入加载模块名和调用函数命->远程调用dlopen函数加载so库->远程调用dlsym函数获取so库中的函数地址->使用ptrace_call远程调用被注入模块的函数->调用dlclose卸载so库->恢复寄存器状态->从远程进程detach(进程暂停->ptrace函数调用，其他函数远程调用->进程恢复)

## 步骤
1. 每个进程都在/proc目录下，以进程id为文件夹名，所以可以通过/proc/<pid>/cmdline文件中中读取进程名称，和我们需要注入的进程名称比较，获得进程id
2. 以root身份运行注入程序，通过ptrace函数，传入PTRACE_ATIACH附加到目标进程，PTRACE_SETREGS设置进程寄存器，PTRACE_GETREGS获得目标寄存器.更多可以访问[ptrace的使用](https://blog.csdn.net/dajian790626/article/details/7781709)
3. 通过ptrace_call函数，调用mmap在对方进程空间分配内存，保存要加载的so文件路径，so中函数的名称，so中函数需要传入的参数
    - 由于每个模块在进程中加载地址不一致，所以我们首先获得目标进程中libc.so文件基址TargetBase，再获得自身libc.so基址SelfBase，再根据mmap-SelfBase+TargetBase获得目标进程中mmap的地址。
    - 同理获得目标进程中dlopen()函数地址、dlsym()函数地址、dlclose()函数地址
4. 通过ptrace_call函数，调用dlopen加载so文件
5. 通过ptrace_call函数，调用dlsym获得so中函数地址
6. 通过ptrace_call函数，调用so中函数
7. 通过ptrace_call函数，调用dlclose卸载so文件
8. 通过ptrace函数，传入PTRACE_DETACH，从目标进程中分离