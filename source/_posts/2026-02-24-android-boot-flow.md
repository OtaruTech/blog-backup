---
title: Android 启动流程详解
date: 2026-02-24 11:30:00
tags:
  - Android
  - 系统启动
  - Bootloader
  - Kernel
  - Init
categories:
  - Android
cover: /img/android-boot-flow.jpg
---

## Android 启动流程详解

Android 系统的启动是一个复杂而精密的过程，涉及多个阶段的协同工作。从按下电源键到看到桌面，整个流程可以分为以下几个关键阶段。

<!--more-->

## 启动流程概览

```
Power On → Boot ROM → Bootloader → Kernel → Init → Zygote → SystemServer → Boot Animation
```

## 1. Boot ROM 阶段

### 1.1 什么是 Boot ROM

Boot ROM 是固化在设备 SoC 中的只读存储器，包含设备启动的第一段代码。这段代码由芯片厂商（如 Qualcomm、MediaTek、Samsung）在出厂时写入，无法修改。

### 1.2 Boot ROM 的主要任务

| 任务 | 说明 |
|------|------|
| 硬件初始化 | 初始化最基本的硬件（CPU、内存控制器） |
| 安全验证 | 验证 Bootloader 的签名（Secure Boot） |
| 加载 Bootloader | 从存储设备（eMMC/UFS）加载 Bootloader 到内存 |

### 1.3 安全启动链

```
Boot ROM (信任根) → 验证 Bootloader → 验证 Kernel → 验证 System 分区
```

这是 Android 安全启动链的第一环，确保只有经过签名的代码才能执行。

## 2. Bootloader 阶段

### 2.1 Bootloader 的作用

Bootloader 是 Android 启动的第二个阶段，常见的 Bootloader 有：
- **LK (Little Kernel)** - Qualcomm 设备常用
- **U-Boot** - 开源设备常用
- **XBL (eXtensible BootLoader)** - Qualcomm 新平台

### 2.2 Bootloader 的主要任务

1. **硬件初始化**：初始化更多硬件（显示屏、存储、USB 等）
2. **加载 Kernel**：从 boot 分区加载 Kernel 和 ramdisk 到内存
3. **传递启动参数**：通过 device tree 或 ATAGs 传递硬件信息给 Kernel
4. **启动模式选择**：根据按键组合进入正常启动、Recovery 或 Fastboot 模式

### 2.3 Bootloader 启动流程

```c
// LK Bootloader 简化流程
void lk_main_app(void) {
    target_early_init();      // 早期硬件初始化
    dload_menu_init();        // 下载模式菜单
    target_late_init();       // 晚期硬件初始化
    boot_linux_from_mmc();    // 从存储加载 Kernel
    start_kernel();           // 跳转到 Kernel 入口
}
```

## 3. Kernel 阶段

### 3.1 Kernel 启动流程

Kernel 启动是 Linux 内核的标准启动流程，主要包含：

1. **汇编入口** (`head.S`)：设置 CPU 模式、创建页表
2. **C 语言入口** (`start_kernel`)：初始化内核子系统
3. **设备驱动初始化**：初始化各种硬件驱动
4. **挂载 rootfs**：挂载 initramfs 作为临时根文件系统
5. **启动 init 进程**：执行 `/init` 程序

### 3.2 Kernel 启动关键日志

```bash
# 查看 Kernel 启动日志
adb shell dmesg | grep -E "Starting|init|Linux version"

# 典型输出
[    0.000000] Linux version 5.10.43-android12-0
[    0.000000] Bootloader: U-Boot 2020.07
[    1.234567] Run /init as init process
```

### 3.3 Android Kernel 特性

| 特性 | 说明 |
|------|------|
| Ashmem | 匿名共享内存，用于进程间通信 |
| Binder | Android 专用 IPC 机制 |
| Alarm | 低功耗定时器 |
| Logger | 系统日志驱动 |
| Power | 电源管理 |

## 4. Init 阶段

### 4.1 Init 进程的作用

Init 是 Kernel 启动的第一个用户空间进程（PID=1），负责：

1. 解析 `init.rc` 配置文件
2. 挂载必要的文件系统
3. 启动核心系统服务
4. 启动 Zygote 进程

### 4.2 Init 配置文件

```rc
# /system/etc/init/hwservicemanager.rc
service hwservicemanager /system/bin/hwservicemanager
    class core
    user system
    group system
    capabilities SYS_NICE
    ioprio rt 4
    writepid /dev/cpuset/system-background/tasks
    onrestart restart android.hidl.allocator@1.0-service
```

### 4.3 Init 启动流程

```
Kernel → /init (PID=1) → 解析 init.rc → 挂载文件系统 → 
启动属性服务 → 启动 Zygote → 启动 SurfaceFlinger
```

## 5. Zygote 阶段

### 5.1 Zygote 的作用

Zygote 是 Android 的核心进程，所有 App 进程都从 Zygote fork 而来：

1. **预加载类库**：加载常用 Java 类到内存
2. **预加载资源**：加载系统资源（图片、字符串等）
3. **启动 SystemServer**：创建系统服务进程
4. **Fork App 进程**：响应 AMS 请求，fork 新的 App 进程

### 5.2 Zygote 启动代码

```java
// frameworks/base/core/jni/com_android_internal_os_ZygoteInit.java
public static void main(String argv[]) {
    ZygoteServer zygoteServer = new ZygoteServer();
    
    // 预加载类和资源
    preload(zygoteServer);
    
    // 启动 SystemServer
    if (startSystemServer) {
        startSystemServer(abiList, socketName, zygoteServer);
    }
    
    // 进入监听循环
    zygoteServer.runSelectLoop(abiList);
}
```

## 6. SystemServer 阶段

### 6.1 SystemServer 的作用

SystemServer 是 Android 系统的核心服务进程，包含所有系统服务：

| 服务 | 作用 |
|------|------|
| ActivityManagerService | 管理 App 生命周期 |
| WindowManagerService | 管理窗口和显示 |
| PackageManagerService | 管理 App 安装和权限 |
| InputManagerService | 管理输入事件 |
| PowerManagerService | 管理电源和唤醒锁 |

### 6.2 SystemServer 启动流程

```java
// frameworks/base/services/java/com/android/server/SystemServer.java
public void run() {
    // 创建系统上下文
    createSystemContext();
    
    // 启动各种系统服务
    startBootstrapServices();
    startCoreServices();
    startOtherServices();
    
    // 进入消息循环
    Looper.loop();
}
```

## 7. Boot Animation 阶段

### 7.1 启动动画

Boot Animation 在 SystemServer 启动过程中显示，直到：

1. SurfaceFlinger 启动完成
2. SystemServer 发送 `BOOT_COMPLETED` 广播
3. Launcher 启动完成

### 7.2 Boot Animation 文件

```
/system/media/bootanimation.zip
├── desc.txt          # 动画描述文件
├── part0/            # 第一部分动画帧
├── part1/            # 第二部分动画帧
└── sound/            # 可选音效
```

## 启动时间优化

### 8.1 启动时间分析工具

```bash
# 查看完整启动时间
adb shell bootstat

# 查看各阶段耗时
adb shell dumpsys boottime

# 查看 App 启动时间
adb shell am start -W <package>/<activity>
```

### 8.2 优化建议

| 阶段 | 优化方法 |
|------|----------|
| Bootloader | 使用 LK 替代 U-Boot，减少初始化代码 |
| Kernel | 裁剪不必要的驱动，使用 initramfs |
| Init | 并行启动服务，减少阻塞 |
| Zygote | 预加载常用类，使用 App Zygote |
| SystemServer | 延迟启动非核心服务 |

## 总结

Android 启动流程是一个多阶段、多进程协作的复杂过程：

1. **Boot ROM** - 硬件级信任根，验证启动链
2. **Bootloader** - 加载 Kernel，传递硬件信息
3. **Kernel** - 初始化硬件，启动 init 进程
4. **Init** - 解析配置，启动核心服务
5. **Zygote** - 预加载资源，fork App 进程
6. **SystemServer** - 启动所有系统服务
7. **Boot Animation** - 显示启动动画直到桌面就绪

理解这个流程对于系统开发、性能优化和问题调试都至关重要。

---

*参考资料：Android Open Source Project, Android System Programming*
