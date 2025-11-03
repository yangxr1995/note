# 环境准备
## 主机准备
```bash
sudo apt update -y
sudo apt install net-tools libncurses5-dev libssl-dev build-essential openssl qemu-system-arm libncurses5-dev gcc-aarch64-linux-gnu git bison flex bc vim universal-ctags cscope cmake python3-dev gdb-multiarch trace-cmd kernelshark bpfcc-tools cppcheck docker docker.io
```

```bash
sudo apt build-dep qemu
wget https://download.qemu.org/qemu-6.2.0.tar.xz
tar -Jxf  qemu-6.2.0.tar.xz
cd qemu-6.2.0
mkdir build
cd build/
../configure --target-list=aarch64-softmmu,riscv64-softmmu
ninja
sudo cp qemu-system-aarch64 /usr/local/bin/qemu-system-aarch64-6.2
sudo cp qemu-system-riscv64 /usr/local/bin/qemu-system-riscv64-6.2
```

## 构建系统
```bash
cd runninglinuxkernel_5.15
./run_rlk_arm64.sh menuconfig
./run_rlk_arm64.sh build_kernel
./run_rlk_arm64.sh build_rootfs
./run_debian_arm64.sh run

./run_rlk_arm64.sh build_kernel         # 重新编译内核
sudo ./run_rlk_arm64.sh update_rootfs  #更新根文件系统
```

## 虚拟机初始化
```bash
# 网络通过 virtIO-Net，所以无需配置，可正常上网
apt update

# 共享目录使用 NET_9P ，需要虚拟机和主机都能使用 NET_9P, 
# 共享目录为runninglinuxkernel_5.15/kmodules
cp test.c  runninglinuxkernel_5.15/kmodules

# 虚拟机安装基本工具
apt install build-essential
```


## 可能遇到的问题
### 证书问题
```bash
apt update
Get:1 http://mirrors.ustc.edu.cn/debian unstable InRelease [165 kB]
Err:1 http://mirrors.ustc.edu.cn/debian unstable InRelease
  The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 648ACFD622F3D138 NO_PUBKEY 0E98404D386FA1D9
Reading package lists... Done
W: GPG error: http://mirrors.ustc.edu.cn/debian unstable InRelease: The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 648ACFD622F3D138 NO_PUBKEY 0E98404D386FA1D9
E: The repository 'http://mirrors.ustc.edu.cn/debian unstable InRelease' is not signed.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
N: See apt-secure(8) manpage for repository creation and user configuration details.

apt-key adv --keyserver pgp.mit.edu --recv-keys 648ACFD622F3D138 //其中648ACFD622F3D138为有问题的证书Key
apt-key adv --keyserver pgp.mit.edu --recv-keys 0E98404D386FA1D9 //其中0E98404D386FA1D9为有问题的证书KEY

apt update
```


# gdb调试
## kernel 配置CONFIG_XX
```bash
CONFIG_PANIC_TIMEOUT=0
# CONFIG_WATCHDOG is not set
# CONFIG_RANDOMIZE_BASE is not set
CONFIG_MAGIC_SYSRQ=y # echo G > /proc/sysrq-trigger
CONFIG_DEBUG_KERNEL=y
CONFIG_DEBUG_INFO=y
CONFIG_DEBUG_INFO_DWARF4=y
CONFIG_FRAME_POINTER=y
CONFIG_GDB_SCRIPTS=y
```

## qemu 启动gdbserver
```bash
qemu -s # 启动gdb
qemu -s -S # 启动gdb并阻塞

./run_rlk_arm64 run debug

zImage 没有符号文件
vmlinux 有符号文件
gdb vmlinux
```

# 查看内核配置
```bash
cat /boot/config-`uname -r` |grep CONFIG_XX
```

# tracefs
## 确保tracefs已存在 
```bash
ls /sys/kernel/tracing/
mount -t tracefs nodev /sys/kernel/tracing/
```

## 
```bash
# 查看可用追踪器
cat /sys/kernel/tracing/available_tracers
blk  - 专门用于块设备追踪，如IO操作
function_graph - 追踪函数入口调用和调用过程
function - 追踪函数入口调用
nop - 禁用追踪功能

# 查看当前运行的追踪器
cat /sys/kernel/tracing/current_tracer

# 启用追踪器
echo 'function_graph' > /sys/kernel/tracing/current_tracer

# 查看追踪日志
# trace是环形缓存
cat /sys/kernel/tracing/trace

# 函数筛选器
## 列出所有可筛选的函数
cat /sys/kernel/tracing/available_filter_functions
## 只追踪某些函数
cat /sys/kernel/tracing/set_ftrace_filter
## 不追踪某些函数
cat /sys/kernel/tracing/set_ftrace_notrace

# 控制环形缓冲区输出
echo 0 > /sys/kernel/tracing/tracing_on
echo function_graph > /sys/kernel/tracing/current_tracer
echo 1 > /sys/kernel/tracing/tracing_on
run_test
echo 0 > /sys/kernel/tracing/tracing_on

# 追踪指定进程
echo $$ > /sys/kernel/tracing/set_ftrace_pid
```

##
trace_printk 比 printk 效率更高，

```bash
echo nop > /sys/kernel/tracing/current_tracer
echo 0 > /sys/kernel/tracing/tracing_on
echo "" > /sys/kernel/tracing/trace
echo 1 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace_pipe
```

# trace-cmd


# trace_printk

# pr_debug
```bash
# 1. 启动输出
# 语法：echo "文件路径 + 匹配规则 + 动作" > /sys/kernel/debug/dynamic_debug/control
# 动作说明：
#   +p：启用打印（print）；-p：禁用打印；
#   +f：显示文件名；+l：显示行号；+m：显示模块名；+t：显示线程ID
# 示例1：启用某驱动文件（如 drivers/usb/core/hub.c）的所有 pr_debug 打印
echo "file drivers/usb/core/hub.c +p" > /sys/kernel/debug/dynamic_debug/control

# 示例2：启用某函数（如 hub_port_init）的 pr_debug 打印，并显示行号和模块名
echo "func hub_port_init +p +l +m" > /sys/kernel/debug/dynamic_debug/control

# 示例3：启用所有文件的 pr_debug 打印（不推荐，日志会过多）
echo "* +p" > /sys/kernel/debug/dynamic_debug/control

# 示例4：查看当前已启用的调试规则
cat /sys/kernel/debug/dynamic_debug/control

# 2. 设置等级
echo 7 > /proc/sys/kernel/printk

# 验证当前日志级别（输出格式：当前级别  默认级别  最低级别  最高级别）
cat /proc/sys/kernel/printk

# 3. 显示日志
# 方法1：实时查看内核日志（推荐，调试时实时观察）
dmesg -wH  # -w：等待新日志；-H：人类可读格式（带时间戳）
# 方法2：查看所有内核日志（包括历史）
dmesg | grep "你的调试关键词"  # 用关键词过滤，避免日志过多
```


