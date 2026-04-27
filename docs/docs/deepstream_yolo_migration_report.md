# DeepStream-Yolo 迁移工作汇报

> 文档定位更新（2026-04-18）  
> 本文是 DeepStream-Yolo 迁移背景与阻塞点记录，不是当前整车主链启动指南。  
> 当前整车主链只依赖 `internal_memory.txt` 这个文件契约；是否真正让 YOLO 结果进入速度仲裁，还取决于 `vth2ros.py -> /cmd_vel_1 -> velpub.py` 这段链路是否打通。

## 工作背景

当前 Jetson 新环境使用的是 `DeepStream 7.1`，而旧小车环境中的 YOLO 插件来自 `DeepStream 6.0/6.1`。  
项目内的控制逻辑，例如 `velpub.py`，依赖 `DeepStream-Yolo` 输出的 `internal_memory.txt`，因此需要把旧环境中带有自定义输出逻辑的 YOLO 插件迁移到新环境。

本次工作的目标是：

- 找到真正用于旧小车运行的 YOLO 插件版本
- 在 `DeepStream 7.1` 上完成插件代码级迁移和编译
- 验证 `deepstream-app` 是否可以正常启动
- 判断旧模型是否还能直接在新环境中继续使用

## 已完成工作

### 1. 确认旧插件版本

已确认真正用于项目运行的老插件目录为：

- `D:/car/DeepStream-Yolo`（我本地文件）

该目录不是普通参考仓库，而是已经带有项目自定义逻辑的版本。  
其中在 `nvdsparsebbox_Yolo.cpp` 中已经存在将识别结果写入 `internal_memory.txt` 的逻辑，这与当前项目运行依赖完全一致。

### 2. 迁移到新环境目录

已将旧插件迁移到 Jetson 新环境目录：

- `/opt/nvidia/deepstream/deepstream-7.1/sources/DeepStream-Yolo`

同时为了兼容旧代码中的硬编码路径与 include 路径，建立了以下兼容软链接：

- `deepstream-6.0 -> deepstream-7.1`
- `deepstream -> deepstream-7.1`

这样可以避免旧代码中写死的 `deepstream-6.0` 路径立即导致运行失败。

### 3. 修复插件编译兼容问题

在重新编译 `nvdsinfer_custom_impl_Yolo` 时，逐步修复了多处由于 DeepStream 版本升级、TensorRT 版本升级以及编译器变化带来的兼容问题，主要包括：

- 多个 layer 源文件缺少 `#include <string>`，导致 `std::string`、`std::stoi`、`std::to_string` 编译失败
- `upsample_layer` 中旧的 TensorRT 枚举 `nvinfer1::ResizeMode::kNEAREST` 需要替换为 `nvinfer1::InterpolationMode::kNEAREST`
- `yolo.cpp` 中旧 TensorRT API `createReorgPlugin` 在当前环境中不存在，已将旧分支封掉，仅保留当前 YOLOv5 使用到的路径
- `Makefile` 中旧链接项 `-lnvparsers` 在当前 TensorRT 环境中不存在，已删除

### 4. 插件重新编译成功

经过上述兼容修复后，已经成功生成：

- `libnvdsinfer_custom_impl_Yolo.so`

这说明旧插件的代码层面已经基本适配 `DeepStream 7.1`。

### 5. 修复 DeepStream 配置文件问题

在运行 `deepstream-app -c deepstream_app_config.txt` 时，先后修复了以下配置问题：

- `[source0]` 段存在非法内联注释，导致配置解析失败
- 摄像头模式 `type=1` 与 `uri=...` 网络流配置混用，导致 `parse_source failed`
- 摄像头设备节点设置错误，`/dev/video1` 不是 capture device，后改为 `/dev/video0`

经过修复后，`deepstream-app` 已经能够正确解析配置文件并进入实际运行阶段。

## 当前状态

当前程序已经可以启动，并进入 engine 重建流程：

- 旧 engine 文件不存在时，DeepStream 已能自动尝试重新构建 engine
- 说明插件加载、配置解析、视频源初始化都已经基本跑通

但在使用当前模型文件构建 YOLO 网络时失败，报错如下：

```text
ERROR: [TRT]: ITensor::getDimensions: Error Code 3: API Usage Error (conv_1: at least 4 dimensions are required for input.)
deepstream-app: utils.cpp:147: int getNumChannels(nvinfer1::ITensor*): Assertion `d.nbDims == 3' failed.
```

## 已排查结论

### 1. 不是类别数配置错误

已核对当前模型相关配置：

- `sign.txt` 内容基本对应 10 个类别
- `yolov5_best.cfg` 中三处 `classes=10`
- 三个检测头 `filters=45`

其关系符合 YOLO 规则：

```text
filters = 3 * (classes + 5) = 3 * (10 + 5) = 45
```

因此可以判断：

- 当前 `classes` 与 `filters` 配置本身是自洽的
- 当前问题不是简单的类别数不匹配

### 2. 当前核心问题已定位

当前的真正阻塞点是：

**旧版 DeepStream-Yolo 中基于 `cfg + wts + native builder` 的 YOLOv5 构图方式，在 `DeepStream 7.1 / TensorRT 10` 下已经不再稳定兼容。**

也就是说：

- 插件已经编译成功
- 配置文件已经能解析
- 摄像头已经能初始化
- 但旧版 YOLOv5 的 `cfg/wts` 构建路线在新环境里失败

## 本次工作成果总结

本次工作已经完成了从“完全无法使用旧 YOLO 插件”到“插件已成功编译并可进入运行阶段”的关键推进，具体成果包括：

- 找到并确认了真正用于旧小车运行的 YOLO 插件版本
- 成功将旧插件迁移到 `DeepStream 7.1` 路径下
- 完成了插件代码级兼容修改并成功编译生成 `.so`
- 解决了 DeepStream 配置文件解析失败问题
- 解决了视频源节点错误问题
- 成功将问题范围缩小并最终定位到模型构建阶段
- 明确排除了类别数配置错误
- 明确判断当前最终阻塞属于旧 YOLOv5 `cfg/wts` 构建路线与新环境不兼容

## 下一步计划

下一步建议不再继续硬修旧版 `cfg + wts + native builder` 路线，而改用 `DeepStream 7.1` 下更稳定的部署方案：

1. 使用新版 `DeepStream-Yolo`
2. 将旧 YOLOv5 模型转换为 `ONNX`
3. 使用新版 `config_infer_primary` 通过 `onnx-file=...` 构建 engine
4. 把旧插件中的 `internal_memory.txt` 输出逻辑移植到新版 `nvdsparsebbox_Yolo.cpp`


