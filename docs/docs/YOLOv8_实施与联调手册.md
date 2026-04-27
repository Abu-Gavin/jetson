# YOLOv8 实施与联调手册

## 1. 文档定位

本文是当前 YOLOv8 迁移工作的统一实施手册。

它覆盖以下内容：

- 独立检测链怎么启动
- YOLO 如何桥接到旧规则层
- 无模型时如何继续联调
- 离线录制与回放怎么用
- 项目模型如何替换
- 当前正式运行命令是什么

从今天起，以下几份文档的具体操作部分统一以本文为准：

- `YOLOv8_独立检测链说明.md`
- `YOLOv8_旁路联调说明.md`
- `YOLOv8_无模型联调说明.md`
- `YOLOv8_离线验证工具说明.md`
- `YOLOv8_项目模型接入说明.md`

## 2. 当前目录与关键文件

### 检测链

- `src/velpub/scripts/yolov8_detector.py`
- `src/velpub/launch/yolov8_detector.launch.py`

### 适配层

- `src/velpub/scripts/yolo_rule_adapter.py`
- `src/velpub/launch/yolo_rule_adapter.launch.py`

### 旧规则桥接

- `src/velpub/scripts/velpub.py`
- `src/velpub/launch/yolov8_to_velpub.launch.py`

### 配置

- `src/velpub/config/yolov8_project_profile.json`
- `src/velpub/config/yolov8_project_profile.template.json`
- `src/velpub/config/yolov8_rule_map.json`
- `src/velpub/config/mock_traffic_sign_sequence.json`

### 工具

- `src/velpub/scripts/validate_yolov8_config.py`
- `src/velpub/scripts/traffic_sign_record.py`
- `src/velpub/scripts/traffic_sign_replay.py`
- `src/velpub/scripts/mock_traffic_sign_detector.py`

## 2.1 YOLO 工作目录与 `.pt` 调用链

当前这套 YOLOv8 检测链的核心工作目录在：

```text
/home/jetson/yahboom_ws/src/velpub
```

其中分工如下：

### 检测节点入口

```text
src/velpub/scripts/yolov8_detector.py
```

作用：

- 读取配置
- 打开摄像头
- 加载 `.pt` 模型
- 发布 `/traffic_sign/detections`
- 提供 YOLO 网页流

### 规则适配层

```text
src/velpub/scripts/yolo_rule_adapter.py
```

作用：

- 订阅 `/traffic_sign/detections`
- 把检测结果映射为旧规则层需要的 `legacy_sign_id`
- 发布 `/traffic_sign/rule_input`

### launch 入口

```text
src/velpub/launch/yolov8_detector.launch.py
src/velpub/launch/yolov8_to_velpub.launch.py
```

### 配置入口

```text
src/velpub/config/yolov8_project_profile.json
src/velpub/config/yolov8_rule_map.json
```

## 2.2 `.pt` 文件是怎么被调用的

当前 `.pt` 的调用链是：

```text
ros2 launch velpub yolov8_detector.launch.py
-> launch 传入 profile_path / model_path
-> yolov8_detector.py 读取 profile
-> 确定最终 model_path
-> ultralytics.YOLO(model_path)
```

当前默认链路如下：

### launch 默认值

文件：

```text
src/velpub/launch/yolov8_detector.launch.py
```

默认参数：

- `profile_path=/home/jetson/yahboom_ws/src/velpub/config/yolov8_project_profile.json`
- `model_path=/home/jetson/yahboom_ws/best.pt`

### profile 默认值

文件：

```text
src/velpub/config/yolov8_project_profile.json
```

当前内容中的核心字段：

```json
"model_path": "/home/jetson/yahboom_ws/best.pt"
```

### 代码实际加载

文件：

```text
src/velpub/scripts/yolov8_detector.py
```

实际通过：

```python
self.model = YOLO(str(model_file))
```

来加载模型。

## 2.3 当前默认模型入口

如果不额外传参，当前默认使用的是：

```text
/home/jetson/yahboom_ws/best.pt
```

所以最常见的替换方式有三种：

### 方式 A：直接替换默认模型文件

覆盖：

```text
/home/jetson/yahboom_ws/best.pt
```

### 方式 B：改 profile

修改：

```text
src/velpub/config/yolov8_project_profile.json
```

里的：

```json
"model_path": "..."
```

### 方式 C：启动时显式覆盖

例如：

```bash
ros2 launch velpub yolov8_detector.launch.py model_path:=/your/path/model.pt
```

## 2.4 当前优先级说明

当前 `model_path` 的优先级可以理解为：

```text
启动命令显式传入的 model_path
> profile 文件里的 model_path
> 代码默认 model_path
```

所以项目组如果后续替换模型，优先建议：

1. 先更新 `yolov8_project_profile.json`
2. 必要时再用 launch 参数临时覆盖

不要长期依赖临时命令行覆盖。

## 3. 当前模型与摄像头分工

### 当前项目模型

默认模型路径：

```text
/home/jetson/yahboom_ws/best.pt
```

当前项目 profile：

```text
/home/jetson/yahboom_ws/src/velpub/config/yolov8_project_profile.json
```

### 当前摄像头分工

- 巡线：`/dev/video0`
- YOLO：`/dev/video2`

### 摄像头回退顺序

YOLO 检测节点当前回退顺序：

```text
/dev/video2,/dev/video0,/dev/video1,/dev/video3
```

## 4. 当前正式命令

### 4.1 只看 YOLO

```bash
source /home/jetson/yahboom_ws/install/setup.bash
ros2 launch velpub yolov8_detector.launch.py show_debug:=false
```

网页：

- `http://<jetson-ip>:8091/`

### 4.2 YOLO 接入旧规则层，但不启底盘

终端 1：

```bash
source /home/jetson/yahboom_ws/install/setup.bash
ros2 launch velpub AIcar_move.launch.py cmd_vel_topic:=/cmd_vel_1
```

终端 2：

```bash
source /home/jetson/yahboom_ws/install/setup.bash
ros2 launch velpub yolov8_to_velpub.launch.py show_debug:=false
```

辅助观察：

```bash
source /home/jetson/yahboom_ws/install/setup.bash
ros2 topic echo /traffic_sign/detections
```

```bash
source /home/jetson/yahboom_ws/install/setup.bash
ros2 topic echo /traffic_sign/rule_input
```

```bash
source /home/jetson/yahboom_ws/install/setup.bash
ros2 topic echo /cmd_vel
```

### 4.3 正式整车启动

终端 1：底盘

```bash
source /home/jetson/yahboom_ws/install/setup.bash
ros2 launch turn_on_wheeltec_robot turn_on_wheeltec_robot.launch.py
```

终端 2：巡线网页与巡线感知

```bash
source /home/jetson/yahboom_ws/install/setup.bash
ros2 launch velpub detect_line.launch.py
```

终端 3：巡线速度输出到 `/cmd_vel_1`

```bash
source /home/jetson/yahboom_ws/install/setup.bash
ros2 launch velpub AIcar_move.launch.py cmd_vel_topic:=/cmd_vel_1
```

终端 4：YOLO + 适配层 + 旧规则层

```bash
source /home/jetson/yahboom_ws/install/setup.bash
ros2 launch velpub yolov8_to_velpub.launch.py show_debug:=false
```

网页：

- 巡线主页面：`http://<jetson-ip>:8090/`
- YOLO 页面：`http://<jetson-ip>:8091/`

## 5. 当前网页结构

### 巡线主页面

地址：

```text
http://<jetson-ip>:8090/
```

内容：

- Original
- Mask
- Fit
- YOLO 卡片

### YOLO 独立页面

地址：

```text
http://<jetson-ip>:8091/
```

内容：

- YOLO 独立流
- 检测框
- 中文类别
- 置信度
- 映射状态
- 右上角状态面板

### 远程访问说明

如果浏览器不能直接访问小车 IP，使用 SSH 端口转发：

```bash
ssh -L 8090:127.0.0.1:8090 -L 8091:127.0.0.1:8091 jetson@<jetson-ip>
```

然后本地浏览器访问：

- `http://127.0.0.1:8090/`
- `http://127.0.0.1:8091/`

## 6. 显示逻辑说明

### 颜色规则

框颜色只表示置信度：

- `>= 0.80`：绿色
- `0.60 ~ 0.79`：黄色
- `< 0.60`：红色

### 文字规则

标签中会显示：

- 类别名
- 置信度
- 是否已映射
- 已映射时显示规则 ID

### 中文显示

当前已改为 `PIL + 中文字体` 绘制，不再使用 `cv2.putText()` 画中文。

## 7. 无模型联调

如果项目模型未就绪，或者想先调规则链，可使用 mock 链。

### 只启动模拟检测器

```bash
source /home/jetson/yahboom_ws/install/setup.bash
ros2 launch velpub mock_traffic_sign_detector.launch.py
```

### 模拟检测直接驱动旧规则层

```bash
source /home/jetson/yahboom_ws/install/setup.bash
ros2 launch velpub mock_to_velpub.launch.py
```

默认模拟序列：

- 红灯
- 空白
- 绿灯
- 空白
- 左转
- 空白
- 限速

## 8. 离线验证

### 录制检测与规则输入

```bash
source /home/jetson/yahboom_ws/install/setup.bash
ros2 run velpub traffic_sign_record.py
```

默认输出：

```text
/home/jetson/yahboom_ws/log/traffic_sign_record.jsonl
```

### 回放检测结果

```bash
source /home/jetson/yahboom_ws/install/setup.bash
ros2 run velpub traffic_sign_replay.py
```

作用：

- 回放 `/traffic_sign/detections`
- 继续调适配层和旧规则层

## 9. 模型替换方法

### 最直接的做法

把新模型覆盖到：

```text
/home/jetson/yahboom_ws/best.pt
```

### 更规范的做法

1. 更新：
   - `src/velpub/config/yolov8_project_profile.json`
2. 必要时更新：
   - `src/velpub/config/yolov8_rule_map.json`

### 替换后先做校验

```bash
source /home/jetson/yahboom_ws/install/setup.bash
ros2 run velpub validate_yolov8_config.py \
  /home/jetson/yahboom_ws/src/velpub/config/yolov8_project_profile.json \
  /home/jetson/yahboom_ws/src/velpub/config/yolov8_rule_map.json
```

## 10. 当前已知特殊点

### 行人逻辑

当前已接入：

- 识别到行人 -> 停车
- 行人移开 -> 恢复进入行人前的状态
- 行人移开时如果当前上游速度仍为 0，则尝试用进入行人前缓存的速度恢复

### 仍需业务确认的类别

- `警告`
- `解除限速`

### 仍需复核的旧逻辑

- 匝道逻辑

## 11. 当前推荐阅读关系

配套主文档：

- [YOLOv8_项目组交接总报告.md](/docs/YOLOv8_项目组交接总报告.md)
- [YOLOv8_迁移总纲_小车现状版.md](/docs/YOLOv8_迁移总纲_小车现状版.md)
- [当前标志类别与控制动作映射表.md](/docs/当前标志类别与控制动作映射表.md)
- [YOLO_规则输入契约说明.md](/docs/YOLO_规则输入契约说明.md)

## 12. 一句话结论

现在关于 YOLOv8 的大多数“怎么启动、怎么接旧规则层、怎么调试、怎么替换模型、怎么回放验证”的内容，都统一以本文为准。
