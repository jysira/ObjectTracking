# 人员目标跟踪：从跑通到评估的三个阶段

本项目使用通用 YOLO 检测器和 ByteTrack，实现固定摄像机人员跟踪。先完成三个阶段，再考虑人体姿态、动作识别。无需先训练新的检测模型。PCB 检测模型不能直接用于检测人。

## 安装与准备（Windows / Anaconda Prompt）

```bat
conda activate industrial_vision
cd /d D:\pycharm\pythonProject\VisualProjects\ObjectTracking
python -m pip install -r requirements.txt
python track.py --help
```

PyTorch 请使用已有的 GPU 环境；CPU 运行时将命令中的 `--device 0` 改成 `--device cpu`。安装依赖需要网络。`yolo11n.pt` 不存在时 Ultralytics 会下载权重；也可将 `--model` 设置为本地通用检测权重的绝对路径。不要使用 PCB 训练后的 best.pt。

准备固定机位、恒定帧率的本地 MP4。先选一段 30～60 秒、2～5 人、有交叉和短暂遮挡的视频，放在 `videos/demo.mp4`。视频请自行准备，仓库不附带。初始演示视频可以短一些，后续另外留出未参与调参的测试视频。不要将同一长视频的相邻片段随机拆成训练与测试素材。

目录：

```text
track.py                 检测、跟踪、轨迹、事件统计
evaluate.py              越线事件与 MOT 身份指标评估
requirements.txt         Python 依赖
docs/01_tracking.md      第一阶段详细说明
docs/02_analytics.md     第二阶段详细说明
docs/03_evaluation.md    第三阶段详细说明
examples/events_gt.csv  人工越线事件表头模板
examples/mot_gt.txt      MOT 真值占位文件（必须自行填写）
videos/                  自备原视频（Git 忽略）
results/                 运行输出（Git 忽略）
reports/                 评估输出（Git 忽略）
```

## 第一阶段：跑通检测与跟踪

```bat
python track.py --source videos/demo.mp4 --model yolo26n.pt --tracker bytetrack.yaml --device 0 --output results/stage1
```

打开 `results/stage` 检查框和 ID，查看 `tracks.csv`。完成标准：能够解释检测器输出和跟踪器关联的区别，能指出漏检、ID 切换和离场重入。详见 [第一阶段](docs/01_tracking.md)。

## 第二阶段：轨迹与业务统计

```bat
python track.py --source videos/demo.mp4 --model yolo26n.pt --tracker bytetrack.yaml --device 0 --line 0.5 0.1 0.5 0.9 --roi 0.2 0.2 0.8 0.9 --output results/stage2
```

坐标均相对原视频宽高归一化。示例线为画面中部竖直线段，ROI 为矩形。请按实际场景调整。输出包括标注视频、逐帧轨迹 `tracks.csv`、事件 `events.csv`、MOT 预测 `mot.txt` 和参数/速度/计数 `summary.json`。

完成标准：人工检查一小段视频，确认越线方向、重复计数和停留时间的含义。详见 [第二阶段](docs/02_analytics.md)。

## 第三阶段：评估与比较

人工填写 `examples/events_gt.csv` 后运行：

```bat
python evaluate.py --output reports/bytetrack_events.json events --prediction results/stage2/events.csv --gt examples/events_gt.csv --tolerance 0.5
```

人工填写完整的 MOT 真值后再运行（空模板不能用于评估）：

```bat
python evaluate.py --output reports/bytetrack_mot.json mot --prediction results/stage2/mot.txt --gt examples/mot_gt.txt --iou 0.5
```

在同一视频和检测参数下对比另一个跟踪器：

```bat
python track.py --source videos/demo.mp4 --model yolo26n.pt --tracker botsort.yaml --device 0 --line 0.5 0.1 0.5 0.9 --roi 0.2 0.2 0.8 0.9 --output results/botsort
python evaluate.py --output reports/botsort_events.json events --prediction results/botsort/events.csv --gt examples/events_gt.csv --tolerance 0.5
python evaluate.py --output reports/botsort_mot.json mot --prediction results/botsort/mot.txt --gt examples/mot_gt.txt --iou 0.5
```

完成标准：在固定测试视频上报告事件 precision/recall、IDF1、ID switches、端到端 FPS，并展示失败案例。详见 [第三阶段](docs/03_evaluation.md)。没有真值时，只能报告速度与定性观察。

## 重要约定

- 每条命令只处理一个视频，新进程从新的跟踪器状态开始。输出目录已存在时自动递增为 `stage1_2`、`stage1_3` 等，不覆盖旧结果；实际目录会打印在终端，并记录在 summary.json 中。
- CSV 的帧从 1 开始，时间从 0 秒开始，坐标是原图像素，位置使用检测框底边中点。
- 默认只检测 COCO 类别 0（person）。自定义模型必须自行指定对应 `--classes`。
- 默认检测置信度 0.1，保留低分框供 ByteTrack 关联；提高阈值可能减少误检，也会加剧断轨。`--iou` 是检测 NMS 参数，不是跟踪关联阈值。
- 跟踪不是身份识别，跨视频、离开画面很久再回来不会保证同一 ID。
- 脚本未训练检测器，未实现姿态估计、自动判断所有错误或 HOTA。完整 HOTA 评测可后续接入官方 TrackEval。
- 运行后的真实数值才可写入简历，示例命令不代表已取得实验成绩。
