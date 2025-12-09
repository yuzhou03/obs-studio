# Windows 捕获插件技术指南

## 1. 概述

OBS Studio 的 `win-capture` 插件是一个专为 Windows 平台设计的高性能视频捕获模块，提供了多种灵活的捕获方式以满足不同场景需求。该插件支持游戏捕获、窗口捕获和显示器捕获等核心功能，并针对不同的 Windows 版本和图形 API 提供了优化的实现策略。

### 1.1 主要功能特点

- **多模式捕获**：支持游戏捕获、窗口捕获和显示器捕获
- **多图形 API 支持**：兼容 DirectX 8/9/10/11/12、OpenGL 和 Vulkan
- **高性能实现**：利用共享纹理和共享内存等技术实现低延迟捕获
- **版本自适应**：根据 Windows 版本自动选择最佳捕获方法
- **兼容性处理**：提供多种兼容性选项以解决特定应用的捕获问题

### 1.2 典型应用场景

- 游戏直播和录制
- 软件演示和教学
- 桌面活动记录
- 远程会议和协作

## 2. 模块架构设计

### 2.1 整体架构

`win-capture` 插件采用模块化设计，由核心功能模块、图形钩子系统和辅助工具三大部分组成：

```
win-capture/
├── 核心功能模块
│   ├── game-capture.c      # 游戏捕获核心实现
│   ├── window-capture.c    # 窗口捕获核心实现
│   ├── monitor-capture.c   # 传统显示器捕获(GDI)
│   └── duplicator-monitor-capture.c  # 高级显示器捕获(DXGI/WGC)
├── 图形钩子系统
│   ├── graphics-hook/      # 图形API钩子实现
│   ├── get-graphics-offsets/  # 图形API偏移量获取
│   └── inject-helper/      # DLL注入辅助工具
└── 辅助工具
    ├── cursor-capture.c    # 鼠标指针捕获
    ├── dc-capture.c        # 设备上下文捕获
    ├── app-helpers.c       # 应用程序辅助函数
    └── audio-helpers.c     # 音频捕获辅助
```

### 2.2 核心组件关系

各组件间通过明确的接口进行交互，形成一个完整的捕获流水线：

1. **插件入口 (`plugin-main.c`)**：负责初始化插件、注册源类型和管理生命周期
2. **捕获源实现**：各类捕获源（游戏、窗口、显示器）实现具体的捕获逻辑
3. **钩子系统**：负责注入目标进程并钩住图形API以获取渲染内容
4. **数据传输层**：使用共享内存或共享纹理技术在进程间传输捕获数据
5. **渲染转换层**：将捕获的数据转换为OBS可使用的纹理格式

### 2.3 设计模式与架构特点

- **插件架构**：遵循OBS插件系统规范，通过注册源信息与OBS核心交互
- **钩子模式**：使用API钩子技术拦截图形渲染调用
- **适配器模式**：为不同图形API提供统一的接口封装
- **工厂模式**：根据配置和环境自动选择最佳捕获方法
- **策略模式**：针对不同场景（如全屏游戏、窗口应用）采用不同的捕获策略

## 3. 核心功能实现原理

### 3.1 游戏捕获 (Game Capture)

游戏捕获是通过向目标游戏进程注入钩子DLL来实现的，它能够直接从游戏的图形API调用中获取渲染内容，是最高效的捕获方式。

#### 3.1.1 实现流程

1. **目标进程识别**：通过窗口标题、窗口类或可执行文件名称识别目标游戏
2. **DLL注入**：使用Windows API将钩子DLL注入目标进程
3. **API钩子安装**：钩住目标进程中的图形API函数（如`SwapBuffers`、`Present`等）
4. **帧捕获**：在图形API调用点获取渲染内容
5. **数据传输**：通过共享内存或共享纹理将捕获的帧传输回OBS进程

#### 3.1.2 核心代码结构

```c
struct game_capture {
    obs_source_t *source;
    obs_source_t *audio_source;
    HWND window;               // 目标窗口句柄
    DWORD process_id;          // 目标进程ID
    ipc_pipe_server_t pipe;    // IPC通信管道
    gs_texture_t *texture;     // 捕获的纹理
    struct hook_info *global_hook_info;  // 钩子信息
    // 其他状态和配置字段
};
```

### 3.2 窗口捕获 (Window Capture)

窗口捕获支持多种捕获方法，包括传统的GDI BitBlt、Windows Graphics Capture (WGC) API等，可根据系统版本和目标窗口特性自动选择最佳方法。

#### 3.2.1 捕获方法对比

| 捕获方法 | 优点 | 缺点 | 适用场景 |
|---------|------|------|--------|
| BitBlt | 兼容性好，实现简单 | 性能较低，不支持硬件加速窗口 | 老旧应用，兼容性要求高 |
| Windows Graphics Capture | 性能好，支持UWP应用 | 仅支持Windows 10 1903+ | 现代应用，需要高帧率捕获 |
| 兼容模式 | 解决特殊窗口问题 | 性能更低 | 特殊应用窗口 |

#### 3.2.2 窗口选择与匹配

窗口捕获支持多种匹配模式，可通过标题、类名或可执行文件名称进行匹配：

```c
enum window_priority {
    PRIORITY_TITLE,  // 优先匹配窗口标题
    PRIORITY_CLASS,  // 优先匹配窗口类
    PRIORITY_EXE,    // 优先匹配可执行文件名
};
```

### 3.3 显示器捕获 (Display Capture)

显示器捕获提供了两种实现方式：基于GDI的传统方式和基于Windows Duplicator API的高级方式，后者提供更好的性能和兼容性。

#### 3.3.1 基于DXGI的Duplicator捕获

```c
struct duplicator_capture {
    obs_source_t *source;
    HMONITOR handle;           // 显示器句柄
    char monitor_id[128];      // 显示器ID
    enum display_capture_method method;  // 捕获方法
    uint32_t width, height;    // 捕获尺寸
    gs_duplicator_t *duplicator;  // DXGI重复器
    // 其他状态字段
};
```

#### 3.3.2 显示器选择与枚举

系统通过`EnumDisplayMonitors` API枚举所有可用显示器，并允许用户选择特定显示器进行捕获：

```c
static BOOL CALLBACK enum_monitor(HMONITOR handle, HDC hdc, LPRECT rect, LPARAM param) {
    struct monitor_info *monitor = (struct monitor_info *)param;
    
    if (monitor->cur_id == 0 || monitor->desired_id == monitor->cur_id) {
        monitor->rect = *rect;
        monitor->id = monitor->cur_id;
    }
    
    UNUSED_PARAMETER(hdc);
    UNUSED_PARAMETER(handle);
    return (monitor->desired_id > monitor->cur_id++);
}
```

## 4. 关键技术选型依据

### 4.1 图形API钩子技术

游戏捕获使用了低级API钩子技术来拦截目标进程的图形渲染调用。这种方法相比其他捕获方式有以下优势：

- **零复制开销**：直接在渲染管道中获取帧数据，无需额外的屏幕抓取
- **支持透明窗口**：能够正确处理具有alpha通道的渲染内容
- **高帧率低延迟**：捕获延迟通常低于其他方法
- **支持硬件加速**：不会干扰目标应用的硬件加速渲染

主要使用的钩子技术包括：

- **Detours库**：用于API函数重定向
- **内存补丁**：直接修改函数入口点
- **IAT钩子**：修改导入地址表

### 4.2 跨进程数据传输

插件使用两种主要机制在捕获进程和OBS主进程之间传输数据：

#### 4.2.1 共享内存 (Shared Memory)

适用于所有Windows版本，但需要额外的同步机制：

```c
struct shmem_data {
    uint32_t cx;            // 宽度
    uint32_t cy;            // 高度
    uint32_t pitch;         // 行间距
    uint32_t format;        // 像素格式
    uint64_t last_updated;  // 最后更新时间戳
    char data[];            // 实际像素数据
};
```

#### 4.2.2 共享纹理 (Shared Texture)

适用于DirectX 11及以上版本，性能更优：

```c
struct shtex_data {
    HANDLE handle;          // 共享纹理句柄
    uint32_t cx;            // 宽度
    uint32_t cy;            // 高度
    uint32_t format;        // 像素格式
    uint32_t share_format;  // 共享格式
    uint64_t last_updated;  // 最后更新时间戳
};
```

### 4.3 Windows版本适配策略

插件根据Windows版本自动选择最佳实现方式：

- **Windows 7/8**：使用GDI或DXGI Duplicator API
- **Windows 10 1903+**：使用Windows Graphics Capture API（性能更好，兼容性更广）

```c
// 检测Windows版本和图形API支持
get_win_ver(&ver);
win8_or_above = ver.major > 6 || (ver.major == 6 && ver.minor >= 2);

obs_enter_graphics();
graphics_uses_d3d11 = gs_get_device_type() == GS_DEVICE_DIRECT3D_11;
obs_leave_graphics();

if (graphics_uses_d3d11)
    wgc_supported = win_version_compare(&ver, &win1903) >= 0;
```

## 5. 数据流程说明

### 5.1 游戏捕获数据流程

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│                 │     │                 │     │                 │
│  目标游戏进程   │────▶│   钩子DLL       │────▶│   OBS主进程     │
│                 │     │                 │     │                 │
└─────────────────┘     └─────────────────┘     └─────────────────┘
         │                       │                       │
         │ 1.渲染帧              │ 3.复制帧数据          │ 5.创建/更新纹理
         ▼                       ▼                       ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│                 │     │                 │     │                 │
│ DirectX/OpenGL  │────▶│ 共享内存/纹理    │────▶│   gs_texture_t  │
│    渲染调用      │     │                 │     │                 │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                              │                       │
                              │ 2.拦截API调用        │ 6.返回给OBS渲染
                              ▼                       ▼
                       ┌─────────────────┐     ┌─────────────────┐
                       │                 │     │                 │
                       │ IPC通信管道      │◀────│  游戏捕获源      │
                       │                 │     │                 │
                       └─────────────────┘     └─────────────────┘
```

### 5.2 窗口捕获数据流程

根据捕获方法的不同，窗口捕获的数据流程有所差异：

#### BitBlt方式

1. 获取目标窗口的设备上下文(HDC)
2. 创建兼容的内存DC和位图
3. 使用BitBlt函数复制窗口内容到内存位图
4. 从内存位图读取像素数据到OBS纹理

#### Windows Graphics Capture方式

1. 创建GraphicsCaptureItem对象，指向目标窗口
2. 创建Direct3D11CaptureFramePool
3. 注册帧到达回调
4. 当帧到达时，通过共享纹理或直接访问获取帧数据
5. 更新OBS纹理

### 5.3 显示器捕获数据流程

#### DXGI Duplicator方式

1. 创建DXGI工厂和适配器
2. 为目标显示器创建OutputDuplication对象
3. 调用AcquireNextFrame获取下一帧
4. 通过共享表面或CPU映射获取像素数据
5. 更新OBS纹理

## 6. API接口定义

### 6.1 OBS源接口

每个捕获功能都实现了OBS源接口，主要包括：

```c
struct obs_source_info {
    const char *id;                 // 源ID
    const char *(*get_name)(void *); // 获取名称回调
    void *(*create)(obs_data_t *);   // 创建源回调
    void (*destroy)(void *);         // 销毁源回调
    uint32_t (*get_width)(void *);   // 获取宽度回调
    uint32_t (*get_height)(void *);  // 获取高度回调
    void (*video_render)(void *);    // 视频渲染回调
    void (*update)(void *, obs_data_t *); // 更新设置回调
    void (*get_defaults)(obs_data_t *);   // 获取默认设置回调
    obs_properties_t *(*get_properties)(void *); // 获取属性回调
    // 其他可选回调...
};
```

### 6.2 游戏捕获核心接口

#### 源注册

```c
extern struct obs_source_info game_capture_info;
```

#### 主要函数

```c
// 创建游戏捕获源
static void *game_capture_create(obs_data_t *settings);

// 渲染游戏捕获内容
static void game_capture_render(void *data);

// 更新游戏捕获设置
static void game_capture_update(void *data, obs_data_t *settings);

// 注入钩子到目标进程
static bool inject_dll(struct game_capture *gc, bool force_32bit);
```

### 6.3 窗口捕获核心接口

#### 源注册

```c
extern struct obs_source_info window_capture_info;
```

#### 主要函数

```c
// 创建窗口捕获源
static void *window_capture_create(obs_data_t *settings);

// 渲染窗口捕获内容
static void window_capture_render(void *data);

// 更新窗口捕获设置
static void window_capture_update(void *data, obs_data_t *settings);

// 获取窗口列表
static bool enum_windows_cb(HWND window, LPARAM param);
```

### 6.4 显示器捕获核心接口

#### 源注册

```c
extern struct obs_source_info monitor_capture_info;
extern struct obs_source_info duplicator_capture_info;
```

#### 主要函数

```c
// 创建显示器捕获源
static void *monitor_capture_create(obs_data_t *settings);
static void *duplicator_capture_create(obs_data_t *settings);

// 渲染显示器捕获内容
static void monitor_capture_render(void *data);
static void duplicator_capture_render(void *data);

// 枚举显示器
static void populate_monitor_list(void *data, obs_property_t *property);
```

## 7. 配置参数说明

### 7.1 游戏捕获配置参数

| 参数名称 | 类型 | 说明 | 默认值 |
|---------|------|------|-------|
| capture_mode | 枚举 | 捕获模式（任意全屏、窗口、热键） | 任意全屏 |
| window | 字符串 | 目标窗口标识 | "" |
| priority | 枚举 | 窗口匹配优先级（标题、类、可执行文件） | 标题 |
| sli_compatibility | 布尔 | SLI兼容性模式 | false |
| capture_cursor | 布尔 | 是否捕获鼠标光标 | true |
| allow_transparency | 布尔 | 允许透明窗口 | false |
| premultiplied_alpha | 布尔 | 使用预乘alpha | false |
| limit_framerate | 布尔 | 限制帧率 | false |
| capture_overlays | 布尔 | 捕获覆盖层 | true |
| anti_cheat_hook | 布尔 | 反作弊兼容钩子 | false |
| hook_rate | 枚举 | 钩子更新频率（慢、正常、快、最快） | 正常 |
| rgb10a2_space | 字符串 | 10位颜色空间（srgb/2100pq） | srgb |

### 7.2 窗口捕获配置参数

| 参数名称 | 类型 | 说明 | 默认值 |
|---------|------|------|-------|
| window | 字符串 | 目标窗口标识 | "" |
| priority | 枚举 | 窗口匹配优先级 | 标题 |
| method | 枚举 | 捕获方法（自动、BitBlt、WGC） | 自动 |
| capture_cursor | 布尔 | 是否捕获鼠标光标 | true |
| compatibility | 布尔 | 兼容性模式 | false |
| client_area | 布尔 | 仅捕获客户端区域 | false |
| force_sdr | 布尔 | 强制SDR模式 | false |

### 7.3 显示器捕获配置参数

| 参数名称 | 类型 | 说明 | 默认值 |
|---------|------|------|-------|
| monitor | 整数 | 显示器索引 | 主显示器 |
| capture_cursor | 布尔 | 是否捕获鼠标光标 | true |
| method | 枚举 | 捕获方法（自动、DXGI、WGC） | 自动 |
| force_sdr | 布尔 | 强制SDR模式 | false |

### 7.4 兼容性配置

`compatibility.json`文件定义了针对特定应用的兼容性设置：

```json
{
    "name": "CS:GO",
    "translation_key": "Compatibility.Application.CSGO",
    "severity": 1,
    "executable": "csgo.exe",
    "game_capture": true,
    "window_capture": false,
    "window_capture_wgc": false,
    "message": "CS:GO may require the <code>-allow_third_party_software</code> launch option to use Game Capture."
}
```

## 8. 性能优化策略

### 8.1 内存管理优化

- **双缓冲机制**：使用两个缓冲区交替读写，减少等待时间
- **零复制传输**：尽可能使用共享纹理避免额外的内存复制
- **按需分配**：仅在需要时分配内存，及时释放不再使用的资源

```c
// 双缓冲机制示例
int next_tex = (data.cur_tex + 1) % NUM_BUFFERS;
// 使用next_tex缓冲区写入新数据
// 切换当前纹理指针
```

### 8.2 图形性能优化

- **格式转换优化**：减少不必要的颜色格式转换
- **批量处理**：合并多次小操作，减少API调用开销
- **硬件加速**：尽可能利用GPU进行数据处理和传输

### 8.3 进程间通信优化

- **高效IPC**：使用命名管道和事件对象进行同步
- **最小化通信量**：仅传输必要的控制信息
- **异步处理**：避免主线程阻塞

```c
static DWORD WINAPI init_hooks(LPVOID param) {
    // 在独立线程中初始化钩子，避免阻塞主线程
    // ...
}

// 创建独立线程初始化钩子
init_hooks_thread = CreateThread(NULL, 0, init_hooks, config_path, 0, NULL);
```

### 8.4 自适应帧率控制

- **动态帧率调整**：根据系统负载和目标帧率调整捕获频率
- **帧率限制**：允许用户限制捕获帧率以减少资源占用
- **智能休眠**：在不需要捕获时暂停处理

## 9. 潜在问题及解决方案

### 9.1 游戏保护冲突

**问题**：许多游戏使用反作弊系统，可能会阻止钩子注入。

**解决方案**：
- 提供反作弊兼容模式选项
- 在兼容性数据库中提供特定游戏的解决方案
- 对于不支持钩子的游戏，自动回退到窗口捕获或显示器捕获

### 9.2 权限不足

**问题**：某些应用可能以较高权限运行，导致无法注入钩子。

**解决方案**：
- 提示用户以管理员权限运行OBS
- 提供替代的捕获方法
- 在UAC提升时保持会话一致性

### 9.3 特殊渲染技术兼容

**问题**：某些游戏使用特殊的渲染技术或多GPU设置（如SLI），可能导致捕获问题。

**解决方案**：
- 提供SLI兼容性选项
- 支持多种图形API和渲染路径
- 自动检测和适应不同的渲染配置

### 9.4 高分辨率高刷新率性能问题

**问题**：捕获4K或更高分辨率、120Hz以上刷新率的内容可能导致性能瓶颈。

**解决方案**：
- 提供可配置的捕获分辨率选项
- 优化内存带宽使用
- 利用硬件加速的缩放和格式转换

### 9.5 DPI缩放兼容性

**问题**：Windows的DPI缩放设置可能导致捕获内容大小不正确。

**解决方案**：
- 使用DPI感知API处理不同的DPI设置
- 正确转换不同DPI环境下的坐标和尺寸

```c
PFN_SetThreadDpiAwarenessContext set_thread_dpi_awareness_context;
PFN_GetThreadDpiAwarenessContext get_thread_dpi_awareness_context;
```

## 10. 总结与最佳实践

### 10.1 捕获方式选择建议

- **游戏直播/录制**：优先使用游戏捕获，提供最佳性能和兼容性
- **一般应用窗口**：使用窗口捕获，可根据系统版本选择BitBlt或WGC
- **全屏内容捕获**：使用显示器捕获，优先选择Duplicator或WGC方法
- **特殊应用**：参考兼容性数据库，选择推荐的捕获方式

### 10.2 性能优化建议

- 仅启用必要的捕获源
- 根据目标平台和带宽限制调整捕获分辨率和帧率
- 对于不活动的源，考虑暂停捕获
- 对于游戏捕获，根据游戏特性调整钩子更新频率

### 10.3 常见问题排查流程

1. **检查目标应用是否在兼容性数据库中**
2. **尝试不同的捕获方法**
3. **检查权限设置，尝试以管理员权限运行**
4. **验证图形驱动是否为最新版本**
5. **对于游戏，检查是否需要特殊启动选项或反作弊兼容设置**

### 10.4 未来发展方向

- 进一步优化对新图形API（如DirectX 12、Vulkan）的支持
- 增强对HDR内容的捕获和处理能力
- 改进跨进程通信机制，进一步降低延迟
- 提供更多针对特定应用和场景的优化选项

通过理解win-capture插件的技术架构和实现细节，开发者可以更好地利用其功能，针对特定场景进行优化，或在需要时扩展其功能以满足更复杂的捕获需求。