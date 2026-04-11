# RKDevelopTool-GUI 项目架构与技术栈

## 1. 项目意图

本项目是一个基于 Python + PySide6 的桌面 GUI 工具，用来为 Rockchip 官方命令行工具 `rkdeveloptool` 提供图形化操作界面。

它的核心目标不是重写底层刷机能力，而是对 `rkdeveloptool` 做一层可视化封装，降低用户直接记忆命令和手工操作的成本，聚焦以下典型场景：

- Rockchip 设备连接检测与模式识别
- 完整固件一键烧录
- 分区表读取与单分区烧录、备份、擦除
- Loader 加载、模式切换、设备重启
- Flash 信息读取、校验、MD5 计算、日志查看
- 多语言与主题/样式切换

从定位上看，它属于：

- 桌面端设备工具
- 图形化外壳应用
- 外部命令驱动型架构

## 2. 总体架构

项目整体是一个“单体桌面应用 + 模块化拆分”的结构，主程序维护全局状态，UI/操作/线程/主题/国际化分别按模块拆开。

```text
RKDevToolGUI(QMainWindow)
|
|-- ui_panels.py
|   负责界面组件和各 Tab 的构建
|
|-- ui_text_updates.py
|   负责多语言切换后的控件文本刷新
|
|-- operations.py
|   负责具体业务动作封装，组织 rkdeveloptool 命令
|
|-- workers.py
|   负责后台线程
|   - 设备轮询
|   - 分区表读取
|   - 长时间命令执行与实时日志
|
|-- utils.py
|   负责工具常量、结果解析、格式化、通用辅助函数
|
|-- themes.py
|   负责主题、样式、系统主题跟随
|
|-- i18n.py
|   负责中英文翻译词典
|
|-- widgets.py
|   负责少量自定义控件扩展
|
`-- 外部依赖: rkdeveloptool
    由 subprocess 调用，实际执行刷机/读取/擦除/切换等底层动作
```

## 3. 分层说明

### 3.1 入口层

入口文件是 `rkdevtoolgui.py`。

这一层负责：

- 创建 `QApplication`
- 校验 `rkdeveloptool` 是否可用
- 初始化翻译管理器 `TranslationManager`
- 创建主窗口 `RKDevToolGUI`
- 维护全局 UI 状态、设备状态、命令执行状态
- 连接退出清理逻辑

这一层是项目的“控制中心”。

### 3.2 表现层

表现层主要由以下文件组成：

- `ui_panels.py`
- `ui_text_updates.py`
- `themes.py`
- `widgets.py`
- `i18n.py`

职责拆分如下：

- `ui_panels.py`
  负责创建左侧设备面板、右侧 Tab 页、日志区，以及大部分按钮事件绑定。
- `ui_text_updates.py`
  负责语言切换后统一刷新窗口标题、分组标题、按钮文字、下拉项等。
- `themes.py`
  负责 Qt 样式切换、深浅色调色板、自动跟随系统主题。
- `widgets.py`
  提供 `AutoLoadCombo`，在下拉展开时触发分区表自动读取。
- `i18n.py`
  以内置字典方式维护中英文文案。

这一层的特点是：

- 以 Qt Widget 为主
- 偏事件驱动
- UI 与业务逻辑没有完全隔离，但已经做了模块化拆分

### 3.3 业务操作层

`operations.py` 是核心业务层，负责把 GUI 操作转换成具体设备动作。

它承担的工作包括：

- 读取设备信息、Flash 信息、安全信息
- 一键烧录、分区烧录、镜像烧录、整机备份
- Loader 加载、模式切换、设备重启
- 分区表读取后的表格填充
- 存储介质识别与切换
- GPT、参数、bootloader 相关高级操作
- 擦除分区、全盘擦除、连接测试

这一层本质上是：

- GUI 和外部命令之间的适配层
- 面向用户动作的用例层

### 3.4 后台执行层

`workers.py` 使用 `QThread` 实现异步任务，避免 GUI 阻塞。

主要线程有：

- `DeviceWorker`
  周期性执行 `rkdeveloptool ld` 检测设备连接状态，并尝试读取芯片信息。
- `PartitionPPTWorker`
  异步执行 `rkdeveloptool ppt` 读取分区表。
- `CommandWorker`
  异步执行任意长时间命令，实时读取 stdout，提取进度百分比并回传日志。

这一层体现了项目的关键设计：

- GUI 主线程只负责交互和显示
- 设备检测与耗时命令放入后台线程
- 通过 Qt Signal/Slot 回传进度、日志和完成状态

### 3.5 工具与解析层

`utils.py` 负责通用能力，主要包括：

- `RKTOOL` 常量定义
- `ToolValidator` 工具可用性校验
- 芯片 ID、Flash 信息、分区信息解析
- 文件 MD5 计算
- 文件大小格式化
- `safe_slot` 信号槽安全包装

这一层支撑了上层业务的“数据解释能力”，把 `rkdeveloptool` 的原始文本输出转成更适合 GUI 展示的数据结构。

## 4. 关键运行流程

### 4.1 应用启动流程

```text
main()
-> 检查 rkdeveloptool 是否存在
-> 创建 QApplication
-> 创建 TranslationManager
-> 创建 RKDevToolGUI
-> 初始化主题/状态栏/左右面板/Tab 页面
-> 启动 DeviceWorker 周期检测设备
```

### 4.2 用户执行操作流程

```text
用户点击按钮
-> ui_panels.py 绑定的事件触发
-> operations.py 组织命令参数
-> RKDevToolGUI.run_command(...)
-> CommandWorker 后台执行 subprocess
-> 日志/进度通过 signal 回到主线程
-> GUI 更新进度条、日志框、状态文本
```

### 4.3 分区管理流程

```text
读取分区表
-> PartitionPPTWorker 执行 rkdeveloptool ppt
-> utils.parse_partition_info() 解析文本
-> operations.py / ui_text_updates.py 回填表格和下拉框
-> 用户可继续执行烧录/备份/擦除
```

### 4.4 设备检测流程

```text
DeviceWorker 周期执行 rkdeveloptool ld
-> 判断设备是否存在
-> 识别 Loader / Maskrom 状态
-> 尝试读取芯片信息
-> 主窗口更新设备列表、连接状态、状态栏提示
```

## 5. 主要模块职责

| 文件 | 角色 | 说明 |
| --- | --- | --- |
| `rkdevtoolgui.py` | 应用入口/主控制器 | 初始化应用、维护状态、协调 UI 与命令执行 |
| `ui_panels.py` | UI 结构定义 | 构建各面板和 Tab，并绑定按钮行为 |
| `operations.py` | 业务动作层 | 封装设备操作、对接 `rkdeveloptool` |
| `workers.py` | 并发执行层 | 后台线程执行设备检测和耗时命令 |
| `utils.py` | 工具与解析层 | 输出解析、文件校验、通用辅助 |
| `ui_text_updates.py` | 国际化刷新层 | 在切换语言时更新所有控件文本 |
| `i18n.py` | 文案资源层 | 中英文本地化字典 |
| `themes.py` | 主题系统 | 深浅色、样式切换、系统主题跟随 |
| `widgets.py` | 自定义控件 | 扩展 Qt 控件行为 |
| `build_nuitka.py` | 构建脚本 | 使用 Nuitka 打包独立可执行程序 |

## 6. 技术栈

### 6.1 语言与运行时

- Python 3.8+

### 6.2 GUI 框架

- PySide6
- Qt Widgets
- Qt Core / Signal / Slot / QThread

### 6.3 设备与系统交互

- `subprocess`
- 外部 CLI：`rkdeveloptool`

### 6.4 并发模型

- Qt `QThread`
- 事件驱动 + 信号槽通信

### 6.5 国际化与主题

- 内置字典式 i18n
- Qt Palette 主题切换
- Qt StyleFactory 样式切换

### 6.6 打包与发布

- Nuitka
- Linux 单文件构建
- macOS `.app` 构建

### 6.7 标准库使用

项目主要使用以下 Python 标准库能力：

- `os`
- `sys`
- `re`
- `hashlib`
- `tempfile`
- `locale`
- `math`
- `platform`
- `pathlib`
- `shutil`

## 7. 外部依赖关系

从实际运行角度看，项目依赖分为两层：

### 7.1 Python 依赖

`requirements.txt` 中的直接依赖：

- `PySide6>=6.5.0`
- `nuitka>=1.8.0`（主要用于打包）

### 7.2 系统级依赖

真正决定功能可用性的关键依赖是：

- `rkdeveloptool`

本项目的大部分核心能力都不是直接在 Python 内实现，而是通过调用 `rkdeveloptool` 实现，所以可以把它理解为：

- Python GUI 外壳
- Rockchip 刷机 CLI 的可视化编排器

## 8. 架构特点总结

这个项目的架构特点比较明确：

- 以桌面 GUI 为中心，而不是库式设计
- 以外部命令封装为核心，而不是直接操作 USB 协议
- 采用模块化单体结构，复杂度适中，易于继续扩展
- 通过 `QThread` 解决耗时命令阻塞问题
- 通过解析文本输出把 CLI 能力转成图形界面能力

如果从工程视角概括，可以定义为：

> 一个基于 PySide6 的、面向 Rockchip 设备刷写与维护场景的图形化命令编排工具。

## 9. 当前项目状态观察

结合仓库现状，可以得到几点补充判断：

- 当前仓库以源码直接运行方式为主，结构简单，没有引入更重的工程框架。
- 没看到独立测试目录或自动化测试配置，当前更偏“可运行工具型项目”。
- 国际化、主题、多 Tab、高级工具等功能已经说明项目不只是最小封装，而是在向“完整桌面设备工具”演进。
- 打包脚本对 macOS 做了额外处理，说明项目有跨平台分发意图。

