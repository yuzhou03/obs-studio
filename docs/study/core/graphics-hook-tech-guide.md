# Graphics-Hook 技术指南

## 1. 概述

Graphics-Hook 是 OBS Studio 中 win-capture 插件的核心组件，负责实现高性能的图形 API 钩子和屏幕捕获功能。它通过 DLL 注入方式，在目标应用程序中钩住各种图形 API（如 DirectX、OpenGL、Vulkan）的关键函数，实现对渲染画面的实时捕获，同时尽量减少对被捕获应用程序性能的影响。

主要功能特点：
- 支持多种图形 API：D3D8、D3D9、D3D10、D3D11、D3D12、OpenGL、Vulkan
- 提供两种捕获模式：共享纹理模式和共享内存模式
- 高性能设计，最小化对游戏/应用性能的影响
- 自动钩子检测和尝试机制
- 线程安全的数据处理流程

## 2. 目录结构

Graphics-Hook 模块采用清晰的功能模块化设计，将不同图形 API 的钩子实现和捕获逻辑分离，同时保持核心逻辑的统一。主要文件组织如下：

```
plugins/win-capture/graphics-hook/
├── CMakeLists.txt              # 构建配置文件
├── graphics-hook.c             # 核心钩子管理和通用功能实现
├── graphics-hook.h             # 核心接口和数据结构定义
├── d3d8-capture.cpp            # D3D8 捕获实现
├── d3d9-capture.cpp            # D3D9 捕获实现
├── d3d9-patches.hpp            # D3D9 钩子补丁定义
├── d3d10-capture.cpp           # D3D10 捕获实现
├── d3d11-capture.cpp           # D3D11 捕获实现
├── d3d12-capture.cpp           # D3D12 捕获实现
├── dxgi-capture.cpp            # DXGI 钩子实现
├── gl-capture.c                # OpenGL 捕获实现
├── vulkan-capture.cpp          # Vulkan 捕获实现
├── vulkan-capture.h            # Vulkan 捕获头文件
├── d3d9-defines.h              # D3D9 相关定义
├── resource.h                  # 资源定义
├── resource.rc                 # 资源文件
```

核心模块分工：
- **graphics-hook.c/h**: 提供钩子管理、进程间通信、共享资源管理等核心功能
- **各 API 捕获文件**: 实现特定图形 API 的钩子和捕获逻辑
- **dxgi-capture.cpp**: 提供对 D3D10/11/12 的底层 DXGI 钩子支持

## 3. 整体架构设计

Graphics-Hook 采用多层架构设计，包括钩子层、捕获层、数据传输层和资源管理层。整体架构设计以最小化性能开销和最大化兼容性为目标。

### 3.1 系统架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         目标应用程序                              │
├───────────────┬─────────────────────────────────────────────────┤
│   钩子检测层   │ 自动检测和尝试各种图形API的钩子                   │
├───────────────┼─────────────────────────────────────────────────┤
│   API钩子层    │ DirectX/OpenGL/Vulkan 渲染函数拦截                │
├───────────────┼─────────────────────────────────────────────────┤
│   捕获实现层   │ 不同API的具体捕获逻辑实现                          │
├───────────────┼─────────────────────────────────────────────────┤
│   数据传输层   │ 共享纹理/共享内存模式的选择与实现                    │
├───────────────┼─────────────────────────────────────────────────┤
│   资源管理层   │ 互斥体、事件、共享内存等系统资源管理                 │
└───────────────┴─────────────────────────────────────────────────┘
                           ↑
                           │ 进程间通信
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│                        OBS Studio 主程序                          │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 核心组件分析

#### 3.2.1 钩子管理模块

钩子管理模块负责初始化钩子环境、检测可钩住的图形 API、尝试安装钩子以及管理钩子的生命周期。核心实现在 `graphics-hook.c` 中的 `attempt_hook()` 函数，它会按照优先级尝试钩住各种图形 API：

```c
static inline bool attempt_hook(void)
{
    /* 按照优先级顺序尝试钩住各种图形 API */
    if (hook_vulkan()) return true;
    if (hook_d3d12()) return true;
    if (hook_dxgi()) return true;  /* 包含 D3D10/11 */
    if (hook_d3d9()) return true;
    if (hook_gl()) return true;
    if (hook_d3d8()) return true;
    if (hook_ddraw()) return true;
    return false;
}
```

#### 3.2.2 捕获模式管理

Graphics-Hook 支持两种主要的捕获模式，根据系统和应用程序的能力自动选择最佳模式：

1. **共享纹理模式**：直接在 GPU 内存中共享纹理，具有更高的性能
2. **共享内存模式**：通过系统共享内存传输纹理数据，兼容性更好

捕获模式的选择逻辑在各个 API 的初始化函数中实现，例如 D3D9 中的：

```cpp
if (global_hook_info->force_shmem || (!d3d9ex && data.patch == -1 && !has_d3d9ex_bool_offset)) {
    success = d3d9_shmem_init(window);
} else {
    success = d3d9_shtex_init(window);
}
```

#### 3.2.3 进程间通信

Graphics-Hook 使用命名管道和共享内存两种机制进行进程间通信：
- 命名管道：用于传递日志和控制命令
- 共享内存：用于传输捕获的帧数据

进程间同步通过 Windows 事件和互斥体实现，确保数据传输的线程安全。

## 4. 核心模块分析

### 4.1 钩子初始化与生命周期管理

Graphics-Hook 的入口点是 DLL_PROCESS_ATTACH，当 DLL 被注入到目标进程时，会执行以下初始化步骤：

1. 检查是否重复注入
2. 初始化信号事件和互斥体
3. 设置系统路径
4. 初始化钩子信息
5. 创建主捕获线程

主捕获线程负责运行捕获循环，不断检测和尝试钩子各种图形 API，其核心逻辑如下：

```c
static inline void capture_loop(void)
{
    WaitForSingleObject(signal_init, INFINITE);

    while (!attempt_hook())
        Sleep(40);  // 尝试钩住图形 API

    // 持续监控，每4秒再次尝试钩子
    for (size_t n = 0; !stop_loop; n++) {
        if (n % 100 == 0)
            attempt_hook();
        Sleep(40);
    }
}
```

### 4.2 API 钩子实现机制

Graphics-Hook 使用 Detours 库来实现函数钩子，主要钩子的图形 API 函数包括：

- D3D9: Present, PresentEx, Reset 等
- D3D11: Present, Present1, ResizeBuffers 等
- OpenGL: SwapBuffers, wglSwapBuffers 等
- Vulkan: vkQueuePresentKHR 等

钩子实现的核心模式是：
1. 获取目标函数的地址
2. 保存原始函数指针
3. 使用 Detours 附加自定义钩子函数
4. 在钩子函数中执行捕获逻辑，然后调用原始函数

以 D3D9 为例，其钩子实现代码片段：

```cpp
bool hook_d3d9(void)
{
    // 获取函数地址
    present_addr = get_offset_addr(d3d9_module, global_hook_info->offsets.d3d9.present);
    
    // 附加钩子
    DetourTransactionBegin();
    RealPresent = (present_t)present_addr;
    DetourAttach((PVOID *)&RealPresent, hook_present);
    DetourTransactionCommit();
    
    return success;
}
```

### 4.3 捕获实现模块

每种图形 API 的捕获实现都遵循类似的模式，但针对特定 API 的特性进行了优化。以 D3D11 为例，其捕获流程如下：

1. 获取后缓冲区指针
2. 根据捕获模式选择合适的捕获方法：
   - 共享纹理模式：创建共享纹理，将后缓冲区内容复制到共享纹理
   - 共享内存模式：创建系统内存表面，将后缓冲区内容复制到系统内存
3. 通过事件通知 OBS 主程序新帧可用

核心捕获逻辑在 `d3d11_capture` 函数中实现：

```cpp
void d3d11_capture(void *swap, void *backbuffer)
{
    // 检查状态
    if (capture_should_stop()) {
        d3d11_free();
    }
    if (capture_should_init()) {
        d3d11_init(swap, backbuffer);
    }
    
    // 执行捕获
    if (data.handle && capture_ready()) {
        if (data.using_shtex)
            d3d11_shtex_capture(backbuffer);
        else
            d3d11_shmem_capture(backbuffer);
    }
}
```

## 5. 数据结构设计

Graphics-Hook 使用了多种关键数据结构来管理捕获状态和数据：

### 5.1 全局钩子信息

```c
extern struct hook_info *global_hook_info;
```

这个结构包含了捕获的配置信息，如格式、大小、帧率等，以及各种 API 函数的偏移量，用于定位钩子函数。

### 5.2 特定 API 数据结构

每种图形 API 都有自己的数据结构，用于管理该 API 特定的资源和状态，例如 D3D9：

```cpp
struct d3d9_data {
    HMODULE d3d9;
    IDirect3DDevice9 *device;
    uint32_t cx, cy;
    D3DFORMAT d3d9_format;
    DXGI_FORMAT dxgi_format;
    bool using_shtex;
    
    // 共享纹理相关资源
    IDirect3DSurface9 *d3d9_copytex;
    ID3D11Device *d3d11_device;
    ID3D11DeviceContext *d3d11_context;
    ID3D11Resource *d3d11_tex;
    
    // 共享内存相关资源
    IDirect3DSurface9 *copy_surfaces[NUM_BUFFERS];
    ID3DQuery9 *queries[NUM_BUFFERS];
    // 其他状态变量...
};
```

### 5.3 线程数据

```c
static struct {
    uint32_t pitch;
    uint32_t cy;
    uint8_t *shmem_textures[2];
    HANDLE copy_event;
    HANDLE stop_event;
    HANDLE copy_thread;
    CRITICAL_SECTION mutexes[NUM_BUFFERS];
    CRITICAL_SECTION data_mutex;
    int cur_tex;
    void *volatile cur_data;
    bool locked_textures[NUM_BUFFERS];
} thread_data;
```

这个结构用于管理共享内存模式下的线程同步和数据传输。

## 6. 数据流程分析

Graphics-Hook 的数据流程是捕获功能的核心，从钩子触发到最终数据传输的完整流程如下：

### 6.1 捕获触发流程

捕获过程通常在目标应用程序执行渲染函数（如 Present）时被触发，以下是典型的数据流程：

1. **钩子触发**：
   - 应用程序调用图形 API 的关键函数（如 `Present`）
   - 自定义钩子函数拦截此调用
   - 钩子函数保存原始调用参数，然后调用捕获函数

2. **状态检查**：
   - 检查捕获状态（是否应该停止/初始化）
   - 如果需要，执行资源初始化或释放

3. **捕获执行**：
   - 获取后缓冲区或渲染目标
   - 根据捕获模式执行相应的捕获操作
   - 更新共享资源状态

4. **通知机制**：
   - 通过事件通知 OBS 主程序新帧可用
   - 调用原始函数，允许应用程序继续正常执行

### 6.2 共享纹理模式数据流

共享纹理模式是性能最优的捕获方式，其数据流如下：

```
应用程序渲染帧 → Present函数钩子 → 创建共享纹理副本 → 
共享纹理写入 → 触发帧就绪事件 → OBS主程序访问共享纹理
```

以 D3D9 共享纹理模式为例，核心实现如下：

```cpp
void d3d9_shtex_capture(IDirect3DSurface9 *surface)
{
    // 将后缓冲区复制到共享纹理
    data.device->StretchRect(surface, NULL, data.d3d9_copytex, NULL,
                          D3DTEXF_NONE);
    
    // 通知OBS主程序新帧可用
    SetEvent(data.notify_event);
}
```

### 6.3 共享内存模式数据流

共享内存模式通过系统内存传输数据，兼容性更好但性能略低：

```
应用程序渲染帧 → Present函数钩子 → 锁定后缓冲区 → 
读取像素数据到系统内存 → 解锁缓冲区 → 
数据复制到共享内存 → 触发帧就绪事件
```

核心实现使用了双重缓冲区和独立线程来减少性能影响：

```c
void shmem_copy_data(uint8_t *data, uint32_t pitch, uint32_t cy, size_t size)
{
    // 锁定数据互斥体
    EnterCriticalSection(&thread_data.data_mutex);
    
    // 更新当前纹理索引，实现双重缓冲
    thread_data.cur_tex = !thread_data.cur_tex;
    thread_data.cur_data = data;
    thread_data.pitch = pitch;
    thread_data.cy = cy;
    
    // 解锁互斥体并触发复制事件
    LeaveCriticalSection(&thread_data.data_mutex);
    SetEvent(thread_data.copy_event);
}
```

## 7. 性能优化策略

Graphics-Hook 实现了多项性能优化策略，以最小化对被捕获应用程序的影响：

### 7.1 异步数据处理

通过独立的复制线程处理纹理数据，避免在渲染线程中进行耗时操作：

```c
static DWORD WINAPI copy_thread(LPVOID param)
{
    uint8_t *shmem_textures[2] = { 0 };
    
    while (WaitForSingleObject(thread_data.stop_event, 0) != WAIT_OBJECT_0) {
        // 等待复制事件
        if (WaitForSingleObject(thread_data.copy_event, 500) != 
                WAIT_OBJECT_0)
            continue;
        
        // 锁定数据互斥体，获取最新纹理
        EnterCriticalSection(&thread_data.data_mutex);
        uint8_t *data = thread_data.cur_data;
        uint32_t pitch = thread_data.pitch;
        uint32_t cy = thread_data.cy;
        int cur_tex = thread_data.cur_tex;
        LeaveCriticalSection(&thread_data.data_mutex);
        
        if (!data || !shmem_textures[cur_tex])
            continue;
        
        // 复制数据到共享内存
        copy_texture_data(data, pitch, shmem_textures[cur_tex], cy);
    }
    
    return 0;
}
```

### 7.2 双重缓冲机制

使用双重缓冲技术减少资源竞争，允许在一个缓冲区分发数据的同时，在另一个缓冲区写入新数据：

```c
uint8_t *shmem_textures[2];  // 双重缓冲区
```

### 7.3 智能钩子尝试

通过优先级排序和间隔尝试，避免频繁失败的钩子操作占用过多CPU资源：

```c
static inline void capture_loop(void)
{
    // ... 初始化代码 ...
    
    // 持续监控，但每4秒才重新尝试一次钩子
    for (size_t n = 0; !stop_loop; n++) {
        if (n % 100 == 0)  // 每100个循环（约4秒）尝试一次
            attempt_hook();
        Sleep(40);  // 40ms间隔，减少CPU使用率
    }
}
```

### 7.4 高效的资源同步

使用互斥体和事件进行线程同步，避免死锁并减少等待时间：

```c
bool try_lock_shmem_tex(int index)
{
    // 尝试锁定互斥体，如果已经被锁定则返回失败（非阻塞操作）
    if (TryEnterCriticalSection(&thread_data.mutexes[index]) == 0)
        return false;
    if (thread_data.locked_textures[index]) {
        LeaveCriticalSection(&thread_data.mutexes[index]);
        return false;
    }
    thread_data.locked_textures[index] = true;
    return true;
}
```

## 8. 关键实现细节

### 8.1 VTable 钩子技术

对于 D3D 接口，Graphics-Hook 使用 VTable 钩子技术，直接修改接口函数的函数指针：

```cpp
// 获取接口的VTable
IDirect3DDevice9Vtbl **device_vtbl = (IDirect3DDevice9Vtbl **)device;

// 保存原始函数指针
OriginalPresent = (*device_vtbl)->Present;

// 替换为自定义钩子函数
(*device_vtbl)->Present = hook_present;
```

### 8.2 动态库加载与函数查找

Graphics-Hook 实现了高效的动态库加载和函数查找机制，支持多种方式获取函数地址：

```c
static inline void *get_module_proc(HMODULE module, const char *func)
{
    return GetProcAddress(module, func);
}

static inline void *get_offset_addr(HMODULE module, ptrdiff_t offset)
{
    return (void *)((uint8_t *)module + offset);
}
```

### 8.3 错误处理与恢复

实现了完善的错误处理机制，能够在捕获失败时自动释放资源并重新尝试：

```c
void d3d11_free(void)
{
    if (!data.using_shtex && data.ready) {
        for (size_t i = 0; i < NUM_BUFFERS; i++) {
            if (data.copy_surfaces[i]) {
                data.copy_surfaces[i]->Release();
                data.copy_surfaces[i] = NULL;
            }
            // 释放其他资源...
        }
    }
    
    // 重置状态
    data.ready = false;
    data.init = false;
    // ...
}
```

### 8.4 渲染穿透检测

实现了智能的渲染穿透检测，避免捕获不相关的渲染操作：

```c
bool should_passthrough(void)
{
    // 检查是否需要跳过当前帧的捕获
    static uint64_t last_capture_time = 0;
    uint64_t current_time = os_gettime_ns();
    
    // 帧率限制逻辑
    if (global_hook_info->fps > 0) {
        uint64_t min_frame_time = 1000000000 / global_hook_info->fps;
        if (current_time - last_capture_time < min_frame_time) {
            return true; // 跳过此帧
        }
    }
    
    last_capture_time = current_time;
    return false;
}
```

## 9. 总结与亮点回顾

Graphics-Hook 是一个设计精巧、性能优化出色的图形API钩子系统，其核心亮点包括：

1. **全面的图形API支持**：覆盖主流图形API，确保广泛的兼容性
2. **灵活的捕获模式**：根据系统和应用程序能力自动选择最佳捕获方式
3. **高性能设计**：通过异步处理、双重缓冲等技术最小化性能开销
4. **健壮的错误处理**：在复杂环境下保持稳定运行
5. **可扩展架构**：模块化设计使得添加新的图形API支持变得简单

Graphics-Hook 通过这些技术实现了高性能、低延迟的游戏捕获，是 OBS Studio 实现高质量录制和直播的关键技术基础。