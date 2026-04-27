# YOLOv8 项目组交接总报告

## 1. 文档目的

本文用于给项目组快速说明：

- 当前迁移做到哪一步
- 我具体改了什么
- 现在工作区里有哪些新增能力
- 项目组下一步应该接着做什么

这份文档优先级高于零散说明文档，项目组接手时建议先读本文。

## 2. 文档收口说明

前面为了快速推进，陆续补了多份 YOLOv8 相关文档。现在已经做收口，建议按以下方式理解：

### 主文档

- [YOLOv8_项目组交接总报告.md](/docs/YOLOv8_项目组交接总报告.md)
- [YOLOv8_迁移总纲_小车现状版.md](/docs/YOLOv8_迁移总纲_小车现状版.md)
- [当前标志类别与控制动作映射表.md](/docs/当前标志类别与控制动作映射表.md)
- [YOLO_规则输入契约说明.md](/docs/YOLO_规则输入契约说明.md)

### 参考/附录文档

- `YOLOv8_长期迁移计划书.md`
- `YOLOv8_独立检测链说明.md`
- `YOLOv8_旁路联调说明.md`
- `YOLOv8_无模型联调说明.md`
- `YOLOv8_离线验证工具说明.md`
- `YOLOv8_项目模型接入说明.md`

也就是说：

- 主文档负责“决策、边界、状态、接手”
- 参考文档负责“具体某个子能力怎么用”

## 3. 当前迁移状态

按照 [YOLOv8_迁移总纲_小车现状版.md](/docs/YOLOv8_迁移总纲_小车现状版.md) 的分层来讲，当前状态如下。

### 第一层：冻结现状

已完成。

已冻结内容：

- 当前主链结构
- 当前规则层输入来源
- 当前旧编码与控制动作映射
- 当前 `internal_memory.txt` 最小兼容语义

### 第二层：独立落地 YOLOv8

已完成“可运行骨架”，并已完成“第一版项目模型接入验证”。

已完成：

- 独立 YOLOv8 检测节点
- 标准检测消息
- 配置化模型/profile/class map 入口
- 已接入项目上传模型 `/home/jetson/yahboom_ws/best.pt`

未完成：

- 项目模型精度与速度验证
- 项目模型类别语义最终确认

### 第三层：建立适配层

已完成第一版可运行适配层。

已完成：

- 标准检测消息到旧规则输入的适配
- `TrafficRuleInput` 消息输出
- 旧 `internal_memory.txt` 兼容输出
- 类别映射配置化

未完成：

- 项目最终类别映射定版
- 稳定性参数的场景化调优
- `警告` / `行人` / `解除限速` 三类业务语义确认

### 第四层：让新检测驱动旧规则层

已完成桥接能力打通，未完成正式联调。

已完成：

- `velpub.py` 支持双输入：
  - 旧文件输入
  - 新 `TrafficRuleInput` 话题输入
- `get_yolo_rst.launch.py` 可切换输入来源
- `yolov8_to_velpub.launch.py` 已存在

未完成：

- 用项目专用模型驱动旧规则层的正式联调
- 关键场景实车验证
- 新模型在真实场景下的类别稳定性验证

### 第五层：整车切主

未开始。

### 第六层：下线历史包袱

未开始。

## 4. 已完成的代码改动

这里只列和 YOLOv8 迁移直接相关的新增或关键改动文件。

## 4.1 新增接口消息

位于 `src/interfaces/msg/`：

- `src/interfaces/msg/TrafficSignDetection.msg`
- `src/interfaces/msg/TrafficSignDetections.msg`
- `src/interfaces/msg/TrafficRuleInput.msg`

同时修改：

- `src/interfaces/CMakeLists.txt`
- `src/interfaces/package.xml`

作用：

- 为新检测链、适配层、规则层桥接提供标准 ROS 接口

## 4.2 新增 YOLOv8 独立检测链

新增：

- `src/velpub/scripts/yolov8_detector.py`
- `src/velpub/launch/yolov8_detector.launch.py`

作用：

- 独立加载 YOLOv8 模型
- 读取摄像头
- 发布 `/traffic_sign/detections`

当前默认模型调用链已固定为：

```text
yolov8_detector.launch.py
-> yolov8_project_profile.json
-> /home/jetson/yahboom_ws/best.pt
-> yolov8_detector.py
-> ultralytics.YOLO(...)
```

当前验证状态：

- 能打开 `/dev/video2`
- 能加载 `/home/jetson/yahboom_ws/best.pt`
- 能启动发布标准消息
- 已验证 `/traffic_sign/detections` 确实有消息输出

## 4.3 新增规则适配层

新增：

- `src/velpub/scripts/yolo_rule_adapter.py`
- `src/velpub/launch/yolo_rule_adapter.launch.py`
- `src/velpub/launch/yolov8_rule_bridge.launch.py`

作用：

- 接收 `/traffic_sign/detections`
- 做类别映射与连续帧确认
- 发布 `/traffic_sign/rule_input`
- 可选写回旧 `internal_memory.txt`

当前验证状态：

- 已验证可以确认 `red_light -> sign_id=5`
- 启动日志能正确打印 `class_map_path`
- 启动日志能正确打印项目 profile 与 fallback 摄像头配置

## 4.4 改造旧规则层接入方式

修改：

- `src/velpub/scripts/velpub.py`
- `src/velpub/launch/get_yolo_rst.launch.py`

变化：

- 默认仍兼容旧 `internal_memory.txt`
- 新增 `use_rule_input_topic` 模式
- 可以从 `/traffic_sign/rule_input` 直接接规则输入

作用：

- 让旧规则层先接受新检测链桥接输入
- 不必第一时间重写旧状态机

## 4.5 新增桥接到旧规则层的组合 launch

新增：

- `src/velpub/launch/yolov8_to_velpub.launch.py`

作用：

- 一键启动：
  - `yolov8_detector.py`
  - `yolo_rule_adapter.py`
  - `velpub.py`

定位：

- 旁路验证新检测如何驱动旧规则层

## 4.6 新增配置化入口

新增：

- `src/velpub/config/yolov8_rule_map.json`
- `src/velpub/config/yolov8_detector_profile.json`
- `src/velpub/config/yolov8_project_profile.template.json`
- `src/velpub/config/yolov8_project_profile.json`
- `src/velpub/scripts/validate_yolov8_config.py`

作用：

- 把类别映射从代码移到配置
- 把检测参数和模型路径从代码移到 profile
- 给项目专用模型接入预留正式入口
- 先做结构校验，避免联调时暴露低级错误

当前验证状态：

- 已跑过校验工具
- 当前 profile 和 rule map 结构有效
- 当前项目 profile 已指向 `/home/jetson/yahboom_ws/best.pt`

## 4.7 新增离线验证工具

新增：

- `src/velpub/scripts/traffic_sign_record.py`
- `src/velpub/scripts/traffic_sign_replay.py`

作用：

- 录制新检测链的输出
- 离线回放 `/traffic_sign/detections`
- 减少每次都上车联调的成本

## 4.8 新增无模型联调链

新增：

- `src/velpub/scripts/mock_traffic_sign_detector.py`
- `src/velpub/config/mock_traffic_sign_sequence.json`
- `src/velpub/launch/mock_traffic_sign_detector.launch.py`
- `src/velpub/launch/mock_to_velpub.launch.py`

作用：

- 在项目模型尚未训练完成前
- 用模拟检测结果先把适配层和旧规则层联调起来

这一步很关键，因为它避免了迁移工作被“模型还没好”卡死。

## 5. 已完成的构建与验证

以下验证已经做过：

1. `interfaces` 包构建通过
2. `velpub` 包构建通过
3. 新消息可通过 `ros2 interface show` 正常读取
4. `yolov8_detector.py` 可正常启动
5. `yolo_rule_adapter.py` 可正常启动
6. `velpub.py` 旧文件输入模式正常
7. `velpub.py` 新 `TrafficRuleInput` 话题输入模式正常
8. 配置校验工具通过：
   - `model_path = /home/jetson/yahboom_ws/best.pt`
   - `mapped_classes = 24`
   - `[OK] configuration files are structurally valid`
9. 项目上传模型类别已读取成功，当前类别为：
   - `右转`
   - `左转`
   - `人行道`
   - `限速`
   - `解除限速`
   - `绿灯`
   - `红灯`
   - `匝道`
   - `警告`
   - `行人`
10. 摄像头 `video0-video3` 已逐一验证：
   - `/dev/video0`：可采集
   - `/dev/video1`：非采集设备
   - `/dev/video2`：可采集
   - `/dev/video3`：非采集设备
11. 检测节点已增加自动回退逻辑：
   - 配置设备失败时，会按 `/dev/video2,/dev/video0,/dev/video1,/dev/video3` 顺序尝试
12. `yolov8_to_velpub.launch.py` 单链启动已验证通过

## 6. 当前最重要的结论

当前工作区已经不再只是“迁移文档”，而是已经具备了下面这条完整迁移骨架：

```text
YOLOv8 检测
-> 标准消息
-> 规则适配层
-> 旧规则层桥接
-> 配置化
-> 离线录制/回放
-> 无模型联调
```

也就是说：

- 迁移已从“规划阶段”进入“工程骨架已落地阶段”
- 当前最大阻塞已经不再是“结构没有搭好”
- 当前环境层面的关键路径已经打通，下一阶段重点转为“类别语义确认”和“规则联调”

## 7. 当前还没完成的部分

以下部分明确还没完成：

1. 项目专用类别表定版
2. 项目专用 rule map 定版
3. `警告` / `行人` / `解除限速` 三类的业务语义确认
4. 用项目模型驱动旧规则层的正式联调
5. 实车关键场景验证
6. 整车切主
7. 下线 `internal_memory.txt`

## 8. 项目组下一步建议

项目组接手后，建议按以下顺序推进。

### 第一步：冻结项目专用类别

项目组需要先统一：

- 模型最终类别名
- 哪些类别进入旧规则层
- 哪些类别先不接规则层
- `警告` 是否映射到旧规则里的 `禁止通行`
- `行人` 是否恢复为旧规则输入
- `解除限速` 是否恢复为旧逻辑输入

产出应落实到：

- 更新 `yolov8_rule_map.json`

### 第二步：接入项目专用模型

项目组模型训练完成后：

1. 准备模型文件
2. 复制并修改：
   - `yolov8_project_profile.template.json`
3. 生成项目实际 profile
4. 用新 profile 跑：
   - `yolov8_detector.launch.py`

### 第三步：先跑新检测链

建议先验证：

```bash
ros2 launch velpub yolov8_detector.launch.py profile_path:=<项目profile>
```

只确认：

- 模型能加载
- 摄像头能跑
- 检测结果合理

### 第四步：再跑检测 + 适配层

建议验证：

```bash
ros2 launch velpub yolov8_rule_bridge.launch.py profile_path:=<项目profile>
```

重点看：

- 类别映射是否正确
- `TrafficRuleInput` 是否稳定
- 旧编码兼容是否符合预期

### 第五步：最后接旧规则层

建议验证：

```bash
ros2 launch velpub yolov8_to_velpub.launch.py profile_path:=<项目profile>
```

重点看：

- 旧 `velpub.py` 的状态切换是否正常
- 关键动作是否合理
- 是否存在误触发或漏触发

### 第六步：善用 mock 链和离线工具

在项目模型还没完全稳定前：

- 用 `mock_to_velpub.launch.py` 先调规则链
- 用 `traffic_sign_record.py` / `traffic_sign_replay.py` 做离线验证

## 9. 建议项目组立即分工

建议接下来由项目组按 3 条线并行推进：

### A. 模型线

负责：

- 训练项目专用 YOLOv8 模型
- 冻结类别名

### B. 接口/适配线

负责：

- 更新 `yolov8_rule_map.json`
- 调适配层确认帧策略

### C. 整车联调线

负责：

- 用 `mock_to_velpub` 和 `yolov8_to_velpub` 推进旧规则层联调
- 留存测试记录

## 10. 一句话交接结论

当前迁移工作的真实状态可以概括成一句话：

**YOLOv8 迁移的工程骨架已经搭起来了，项目组接下来不需要再从零设计结构，重点转为“接入项目模型、定版映射、跑联调”。**
