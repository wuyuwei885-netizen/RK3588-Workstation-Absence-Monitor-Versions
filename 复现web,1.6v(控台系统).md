# 复现web,1\.6v\(控台系统\)

# 二、复现所需文件

## 项目源码包

Windows 电脑上准备：

```Plain Text
C:\Users\PC\Downloads\workstation_absence_v1.0.6.1_full_fixed.zip
```

## RK3588 板端三个模型

必须存在：

```Plain Text
/root/yolov8_rk3588/code/model/yolov8n-pose-int8.rknn
/root/yolov8_rk3588/code/model/RetinaFace_mobile320-int8.rknn
/root/yolov8_rk3588/code/model/w600k_mbf-int8-calib500.rknn
```

三个模型分别使用 NPU Core 0、Core 1、Core 2；人脸识别模型输入为 RGB、uint8、NHWC，并且 Python 端不重复归一化。

## 已有 Python 环境

继续使用：

```Plain Text
/root/yolov8_rk3588/rknn310
```

不要重新创建虚拟环境，也不要重新安装 OpenCV、NumPy、RKNNLite。

## 摄像头

默认：

```Plain Text
/dev/video22
```

## 项目关键目录结构

解压后至少应存在：

```Plain Text
/root/workstation_absence/
├── app.py
├── config.example.json
├── requirements-rk3588.txt
├── workstation/
│   ├── camera.py
│   ├── config.py
│   ├── database.py
│   ├── enrollment.py
│   ├── identity.py
│   ├── models.py
│   ├── presence.py
│   ├── runtime.py
│   ├── tracking.py
│   └── web.py
├── templates/
├── static/
├── tools/
├── tests/
├── systemd/
└── data/
```

项目基线目录结构及五个网页页面在 README 中有明确记录。

---

# 三、具体复现步骤

## 第一步：Windows 校验源码压缩包

打开 Windows CMD：

```Plain Text
cd /d C:\Users\PC\Downloads

dir workstation_absence_v1.0.6.1_full_fixed.zip

certutil -hashfile workstation_absence_v1.0.6.1_full_fixed.zip SHA256
```

正确结果必须包含：

```Plain Text
0d5378b0b6c15991b102e2c6fd434916bc0fd266face9d00fb128b0b52baf30d
```

不一致说明：

```Plain Text
文件损坏
下载不完整
拿错版本
被重新压缩过
```

---

## 第二步：上传到 RK3588

板子 IP 按你现在的地址：

```Plain Text
192.168.10.66
```

Windows CMD 执行：

```Plain Text
scp "C:\Users\PC\Downloads\workstation_absence_v1.0.6.1_full_fixed.zip" root@192.168.10.66:/root/
```

上传完成后登录板子：

```Plain Text
ssh root@192.168.10.66
```

---

## 第三步：停止旧程序并备份

以下命令在 RK3588 执行。

先停止 systemd 服务，防止服务自动重新占用摄像头：

```Plain Text
systemctl stop workstation-absence 2>/dev/null || true
systemctl disable workstation-absence 2>/dev/null || true
```

停止旧 Python 程序：

```Plain Text
pkill -f '/root/workstation_absence.*app.py' 2>/dev/null || true
pkill -f 'python.*app.py.*config.json' 2>/dev/null || true
pkill -f 'v4l2-ctl.*video22' 2>/dev/null || true
```

检查 8090 端口：

```Plain Text
fuser -v 8090/tcp 2>/dev/null || true
```

检查摄像头占用：

```Plain Text
fuser -v /dev/video22 2>/dev/null || true
```

备份旧项目：

```Plain Text
cd /root

STAMP="$(date +%Y%m%d_%H%M%S)"

if [ -d /root/workstation_absence ]; then
    mv /root/workstation_absence "/root/workstation_absence.backup.${STAMP}"
    echo "旧项目已备份到：/root/workstation_absence.backup.${STAMP}"
else
    echo "没有发现旧项目目录"
fi
```

查看备份：

```Plain Text
ls -ld /root/workstation_absence.backup.* 2>/dev/null || true
```

---

## 第四步：板端再次验证 ZIP

```Plain Text
echo "0d5378b0b6c15991b102e2c6fd434916bc0fd266face9d00fb128b0b52baf30d  /root/workstation_absence_v1.0.6.1_full_fixed.zip" | sha256sum -c -
```

正常输出：

```Plain Text
/root/workstation_absence_v1.0.6.1_full_fixed.zip: OK
```

如果显示：

```Plain Text
FAILED
```

不要继续解压。

---

## 第五步：解压到固定目录

为了避免板子没有 `unzip`，直接使用现有 Python 环境解压：

```Plain Text
rm -rf /root/workstation_restore_tmp
mkdir -p /root/workstation_restore_tmp

/root/yolov8_rk3588/rknn310/bin/python -m zipfile -e \
/root/workstation_absence_v1.0.6.1_full_fixed.zip \
/root/workstation_restore_tmp
```

查找项目根目录：

```Plain Text
find /root/workstation_restore_tmp -type f -name app.py -print
```

自动复制到固定路径：

```Plain Text
PROJECT_ROOT="$(dirname "$(find /root/workstation_restore_tmp -type f -name app.py | head -n 1)")"

if [ -z "$PROJECT_ROOT" ] || [ ! -f "$PROJECT_ROOT/app.py" ]; then
    echo "错误：压缩包中没有找到 app.py"
    exit 1
fi

mkdir -p /root/workstation_absence
cp -a "$PROJECT_ROOT"/. /root/workstation_absence/

rm -rf /root/workstation_restore_tmp
```

检查：

```Plain Text
cd /root/workstation_absence

test -f app.py && echo "app.py OK"
test -f config.example.json && echo "config.example.json OK"
test -f requirements-rk3588.txt && echo "requirements-rk3588.txt OK"
test -d workstation && echo "workstation目录 OK"
test -d templates && echo "templates目录 OK"
test -d static && echo "static目录 OK"
test -d tools && echo "tools目录 OK"
```

查看完整结构：

```Plain Text
find /root/workstation_absence -maxdepth 2 -type f | sort
```

---

## 第六步：生成 v1\.0\.6\.1 配置

进入项目：

```Plain Text
cd /root/workstation_absence
```

使用基线配置：

```Plain Text
rm -f config.json
cp config.example.json config.json
```

基线配置已经定义：

```Plain Text
Web端口：8090
摄像头：/dev/video22
分辨率：1920×1080
像素格式：NV12
Pose模型：NPU Core 0
RetinaFace：NPU Core 1
人脸识别：NPU Core 2
```

完整配置内容已经明确记录。

执行下面的 Python 命令，强制修正路径，避免旧压缩包内路径不一致：

```Plain Text
source /root/yolov8_rk3588/rknn310/bin/activate && cd /root/workstation_absence && python - <<'PY'
import json
from pathlib import Path

path = Path("config.json")

with path.open("r", encoding="utf-8") as f:
    cfg = json.load(f)

cfg["app"]["host"] = "0.0.0.0"
cfg["app"]["port"] = 8090
cfg["app"]["data_dir"] = "/root/workstation_absence/data"
cfg["app"]["database"] = "/root/workstation_absence/data/workstation.db"
cfg["app"]["enrollment_dir"] = "/root/workstation_absence/data/face_data"
cfg["app"]["event_dir"] = "/root/workstation_absence/data/events"

cfg["camera"]["device"] = "/dev/video22"
cfg["camera"]["width"] = 1920
cfg["camera"]["height"] = 1080
cfg["camera"]["fps"] = 20
cfg["camera"]["pixel_format"] = "NV12"

cfg["models"]["pose"]["enabled"] = True
cfg["models"]["pose"]["path"] = (
    "/root/yolov8_rk3588/code/model/yolov8n-pose-int8.rknn"
)
cfg["models"]["pose"]["core"] = "0"

cfg["models"]["retinaface"]["enabled"] = True
cfg["models"]["retinaface"]["path"] = (
    "/root/yolov8_rk3588/code/model/"
    "RetinaFace_mobile320-int8.rknn"
)
cfg["models"]["retinaface"]["core"] = "1"

cfg["models"]["face_recognition"]["enabled"] = True
cfg["models"]["face_recognition"]["path"] = (
    "/root/yolov8_rk3588/code/model/"
    "w600k_mbf-int8-calib500.rknn"
)
cfg["models"]["face_recognition"]["core"] = "2"
cfg["models"]["face_recognition"]["normalize"] = False
cfg["models"]["face_recognition"]["similarity_threshold"] = 0.55

with path.open("w", encoding="utf-8") as f:
    json.dump(cfg, f, ensure_ascii=False, indent=2)

print("config.json 已生成")
PY
```

检查 JSON：

```Plain Text
source /root/yolov8_rk3588/rknn310/bin/activate && cd /root/workstation_absence && python -m json.tool config.json >/dev/null && echo "config.json OK"
```

查看关键配置：

```Plain Text
grep -nE '8090|video22|yolov8n-pose|RetinaFace|w600k|normalize|similarity_threshold' config.json
```

---

## 第七步：检查三个 RKNN 模型

执行：

```Plain Text
for MODEL in \
/root/yolov8_rk3588/code/model/yolov8n-pose-int8.rknn \
/root/yolov8_rk3588/code/model/RetinaFace_mobile320-int8.rknn \
/root/yolov8_rk3588/code/model/w600k_mbf-int8-calib500.rknn
do
    if [ -s "$MODEL" ]; then
        echo "[OK] $MODEL"
        ls -lh "$MODEL"
    else
        echo "[ERROR] 模型不存在或大小为0：$MODEL"
    fi
done
```

计算模型校验值并保存：

```Plain Text
mkdir -p /root/workstation_absence/data

sha256sum \
/root/yolov8_rk3588/code/model/yolov8n-pose-int8.rknn \
/root/yolov8_rk3588/code/model/RetinaFace_mobile320-int8.rknn \
/root/yolov8_rk3588/code/model/w600k_mbf-int8-calib500.rknn \
| tee /root/workstation_absence/data/model_sha256.txt
```

---

## 第八步：检查摄像头

```Plain Text
ls -l /dev/video22
```

检查支持格式：

```Plain Text
v4l2-ctl -d /dev/video22 --all
```

```Plain Text
v4l2-ctl -d /dev/video22 --list-formats-ext
```

检查是否被其他程序占用：

```Plain Text
fuser -v /dev/video22 2>/dev/null || true
```

正常情况下，在项目启动前不应有旧的 `python` 或 `v4l2-ctl` 持有该设备。

---

## 第九步：激活环境和安装 Web 依赖

先确认 Python：

```Plain Text
source /root/yolov8_rk3588/rknn310/bin/activate

which python
python -V
```

正常应类似：

```Plain Text
/root/yolov8_rk3588/rknn310/bin/python
Python 3.10.x
```

进入项目：

```Plain Text
cd /root/workstation_absence
export PYTHONPATH=/root/workstation_absence
```

先检查已有依赖：

```Plain Text
python -c "import cv2,numpy; from rknnlite.api import RKNNLite; print('OpenCV:',cv2.__version__); print('NumPy:',numpy.__version__); print('RKNNLite OK')"
```

检查 Web 依赖：

```Plain Text
python -c "import fastapi,uvicorn,jinja2,multipart; print('Web dependencies OK')"
```

只有上面出现模块缺失时，才执行：

```Plain Text
source /root/yolov8_rk3588/rknn310/bin/activate && cd /root/workstation_absence && python -m pip install -r requirements-rk3588.txt
```

再次检查：

```Plain Text
source /root/yolov8_rk3588/rknn310/bin/activate && cd /root/workstation_absence && python -c "import cv2,numpy,fastapi,uvicorn,jinja2,multipart; from rknnlite.api import RKNNLite; print('ALL IMPORTS OK')"
```

---

## 第十步：语法和项目自测

完整语法检查：

```Plain Text
source /root/yolov8_rk3588/rknn310/bin/activate && cd /root/workstation_absence && export PYTHONPATH=/root/workstation_absence && python -m py_compile app.py workstation/*.py tools/*.py tests/*.py
```

没有任何输出即为成功。

运行自测：

```Plain Text
source /root/yolov8_rk3588/rknn310/bin/activate && cd /root/workstation_absence && export PYTHONPATH=/root/workstation_absence && python tools/run_self_test.py
```

正常最后输出：

```Plain Text
ALL AVAILABLE SELF TESTS PASSED
```

v1\.0\.6 审计记录中，配置、数据库、追踪、删除/重置、身份绑定、摄像头恢复、模板和 Web 集成测试均通过；但真实摄像头和 NPU 仍需要在 RK3588 上验证。

如果压缩包没有 `run_self_test.py`，分别执行：

```Plain Text
source /root/yolov8_rk3588/rknn310/bin/activate && cd /root/workstation_absence && export PYTHONPATH=/root/workstation_absence && python tests/test_database.py
```

```Plain Text
source /root/yolov8_rk3588/rknn310/bin/activate && cd /root/workstation_absence && export PYTHONPATH=/root/workstation_absence && python tests/test_tracker.py
```

---

## 第十一步：板端环境检查

优先执行：

```Plain Text
source /root/yolov8_rk3588/rknn310/bin/activate && cd /root/workstation_absence && export PYTHONPATH=/root/workstation_absence && python tools/check_environment.py --config config.json
```

如果存在更完整的检查工具：

```Plain Text
source /root/yolov8_rk3588/rknn310/bin/activate && cd /root/workstation_absence && export PYTHONPATH=/root/workstation_absence && if [ -f tools/board_health_check.py ]; then python tools/board_health_check.py --config config.json; fi
```

正常应出现：

```Plain Text
Architecture: aarch64
v4l2-ctl: /usr/bin/v4l2-ctl
RKNNLite: import OK
pose: OK
retinaface: OK
face_recognition: OK
Camera: /dev/video22
```

这些也是原项目文档定义的板端通过条件。

---

## 第十二步：初始化干净数据

这是全新复现时执行：

```Plain Text
source /root/yolov8_rk3588/rknn310/bin/activate && cd /root/workstation_absence && export PYTHONPATH=/root/workstation_absence && mkdir -p data/face_data data/events && python tools/init_demo_data.py --config config.json
```

如果显示数据已经存在，不需要重复初始化。

---

# 四、一条完整命令启动

这是你以后固定使用的启动命令：

```Plain Text
source /root/yolov8_rk3588/rknn310/bin/activate && cd /root/workstation_absence && export PYTHONPATH=/root/workstation_absence && mkdir -p data && python app.py --config config.json 2>&1 | tee /root/workstation_absence/data/app_test.log
```

该命令同时完成：

```Plain Text
激活 rknn310
进入项目目录
设置 PYTHONPATH
创建日志目录
启动 app.py
保存完整日志
```

这也是项目文档记录的固定启动上下文，可以避免：

```Plain Text
No module named workstation
解释器用错
相对路径错误
日志丢失
```



---

# 五、后台运行命令

确认前台运行稳定后，按 `Ctrl+C` 停止，再执行：

```Plain Text
cd /root/workstation_absence && mkdir -p data && nohup env PYTHONPATH=/root/workstation_absence /root/yolov8_rk3588/rknn310/bin/python app.py --config config.json > /root/workstation_absence/data/app.log 2>&1 & echo $!
```

查看进程：

```Plain Text
pgrep -af 'app.py.*config.json'
```

查看日志：

```Plain Text
tail -f /root/workstation_absence/data/app.log
```

停止：

```Plain Text
pkill -f '/root/workstation_absence.*app.py'
```

---

# 六、验证方法

## 验证端口

```Plain Text
ss -lntp | grep ':8090'
```

正常类似：

```Plain Text
LISTEN 0 2048 0.0.0.0:8090
```

## 验证首页

```Plain Text
curl -I http://127.0.0.1:8090/
```

正常：

```Plain Text
HTTP/1.1 200 OK
```

## 验证状态接口

```Plain Text
curl -s http://127.0.0.1:8090/api/status | python -m json.tool
```

重点检查：

```Plain Text
camera_online
runtime_alive
model_errors
active_shifts
```

正常要求：

```Plain Text
camera_online = true
runtime_alive = true
model_errors = {}
```

## 自动页面测试

```Plain Text
source /root/yolov8_rk3588/rknn310/bin/activate && cd /root/workstation_absence && export PYTHONPATH=/root/workstation_absence && if [ -f tools/api_smoke_test.py ]; then python tools/api_smoke_test.py --base-url http://127.0.0.1:8090; fi
```

测试计划要求五个页面、`/api/status` 和 CSV 接口通过，且 `model_errors` 为空。

## 浏览器访问

电脑或者手机打开：

```Plain Text
http://192.168.10.66:8090/
```

预计页面：

```Plain Text
员工管理
工作台配置
排班配置
实时监控
历史记录
```



