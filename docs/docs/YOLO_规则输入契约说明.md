# YOLO 规则输入契约说明

## 1. 文档定位

本文定义“交通标志检测链向规则仲裁层提供输入”的契约。

它的作用是：

- 固定当前旧接口的最低兼容要求
- 定义迁移期的双接口方案
- 定义未来标准 ROS 接口的基础字段
- 明确 YOLOv8 适配层应该产出什么

后续所有检测模型、适配器、规则层输入改造，都必须遵守本文。

## 2. 当前旧接口契约

当前规则层 `velpub.py` 读取的旧接口为：

```text
/opt/nvidia/deepstream/deepstream-6.0/sources/DeepStream-Yolo/nvdsinfer_custom_impl_Yolo/internal_memory.txt
```

### 2.1 文件格式

当前最小兼容格式为两行：

```text
<sign>
<area>
```

示例：

```text
5
0.0184
```

其中：

- 第 1 行 `sign`
  - 当前规则层真正依赖的核心字段
- 第 2 行 `area`
  - 当前保留兼容，但大部分逻辑未真正消费

### 2.2 语义要求

- `sign` 为空字符串时，表示当前没有有效标志输入
- `area` 当前可写 `0.0`
- 文件写入必须覆盖式写入，避免旧值残留

### 2.3 当前必须兼容的编码

迁移期必须至少兼容以下编码：

- `0`：人行道
- `2`：匝道
- `3`：绿灯恢复
- `4`：左转
- `5`：红灯
- `7`：减速
- `8`：禁止通行
- `100`：右转

## 3. 迁移期接口策略

迁移期采用“双接口并存”策略：

```text
YOLOv8 原始检测
-> 适配层
-> internal_memory.txt（旧接口兼容）
-> ROS 标准消息（新接口）
```

这样做的目的：

- 不破坏当前 `velpub.py`
- 给新规则层和调试工具提供标准消息
- 保证可以逐步切换，而不是一次性替换

## 4. 新标准接口定义

本项目已在 `interfaces` 包中增加以下消息：

### 4.1 `interfaces/msg/TrafficSignDetection`

用于表达单个检测框的标准化结果。

字段：

- `std_msgs/Header header`
- `string class_name`
- `int32 legacy_sign_id`
- `float32 confidence`
- `int32 xmin`
- `int32 ymin`
- `int32 xmax`
- `int32 ymax`
- `float32 area_ratio`
- `bool confirmed`
- `string source`

设计意图：

- `class_name` 保留模型原始类别语义
- `legacy_sign_id` 提供到旧规则层的映射出口
- `area_ratio` 为旧接口和后续规则判断保留面积语义
- `confirmed` 表示是否已经通过适配层的连续帧确认
- `source` 用于标记来自哪个检测链

### 4.2 `interfaces/msg/TrafficSignDetections`

用于表达一帧中的检测结果集合。

字段：

- `std_msgs/Header header`
- `interfaces/TrafficSignDetection[] detections`

设计意图：

- 用于新检测链输出原始或半处理后的结果
- 作为适配层输入

### 4.3 `interfaces/msg/TrafficRuleInput`

用于表达最终送入规则层的标准化输入。

字段：

- `std_msgs/Header header`
- `int32 legacy_sign_id`
- `string class_name`
- `float32 confidence`
- `float32 area_ratio`
- `bool confirmed`
- `string source`

设计意图：

- 表达“规则层真正需要消费的最终结果”
- 比原始检测消息更稳定、更接近控制层语义

## 5. 适配层职责契约

适配层不是简单“转个格式”，而是负责把 YOLOv8 输出变成规则层可消费的稳定输入。

### 5.1 输入

适配层输入应为：

- `interfaces/msg/TrafficSignDetections`

默认话题建议：

```text
/traffic_sign/detections
```

### 5.2 输出

适配层必须同时支持：

1. 写旧接口：
   - `internal_memory.txt`
2. 发新接口：
   - `interfaces/msg/TrafficRuleInput`

默认话题建议：

```text
/traffic_sign/rule_input
```

### 5.3 行为要求

适配层必须负责：

- 类别名到旧编码映射
- 最低置信度过滤
- 连续帧确认
- 超时清空旧标志
- 输出源标记

适配层不负责：

- 巡线控制
- 底盘控制
- 最终规则状态机执行

## 6. 已实现的过渡骨架

当前工作区中已经新增过渡骨架：

- 脚本：
  - `src/velpub/scripts/yolo_rule_adapter.py`
- launch：
  - `src/velpub/launch/yolo_rule_adapter.launch.py`

其定位是：

- 接收标准检测消息
- 通过映射与连续确认生成规则输入
- 同时兼容旧 `internal_memory.txt`

当前默认映射包括：

- `crosswalk` -> `0`
- `ramp` -> `2`
- `green` / `green_light` -> `3`
- `left` / `turn_left` -> `4`
- `red` / `red_light` -> `5`
- `slow` / `slow_down` -> `7`
- `no_entry` -> `8`
- `right` / `turn_right` -> `100`

注意：

- 这只是第一版兼容映射
- 最终类别表仍要以项目组冻结结果为准

## 7. 迁移阶段的强制要求

在旧规则层未被替换前，任何新检测链都必须满足：

1. 能产出与旧编码兼容的 `legacy_sign_id`
2. 能输出稳定的确认结果，而不是单帧抖动结果
3. 能在无结果超时后清空旧状态
4. 不能要求 `velpub.py` 立即重写才可运行

## 8. 未来正式切换目标

长期目标是：

- `velpub.py` 或其后继规则仲裁节点不再直接读 `internal_memory.txt`
- 规则层直接消费标准 ROS 消息
- `internal_memory.txt` 仅在迁移过渡期存在

当以下条件满足时，才允许下线旧接口：

1. 新检测链稳定
2. 适配层稳定
3. 规则层已能稳定消费标准消息
4. 实车联调完成

## 9. 结论

当前规则输入契约的核心不是“更多字段”，而是：

```text
稳定类别语义
+ 旧编码兼容
+ 连续帧确认
+ 可观测的标准消息
```

因此，在 YOLOv8 迁移过程中：

- `internal_memory.txt` 是迁移兼容层，不是长期目标
- `TrafficSignDetections` 和 `TrafficRuleInput` 是未来正式接口的基础
- `yolo_rule_adapter.py` 是新旧接口之间的过渡桥

这就是后续检测链与规则链对接时必须遵守的输入契约。
