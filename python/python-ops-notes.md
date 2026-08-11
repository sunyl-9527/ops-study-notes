# Python 运维开发笔记
> 适用对象：已有 Python 基础语法，见 [学习路线.md](./学习路线.md)。
>
> 定位：把 Python 用到运维自动化场景，是从运维走向运维开发的核心技能。
本文不再重复变量、循环、函数、类等基础语法，而是围绕文件处理、命令执行、接口调用、远程操作、任务调度和工具交付展开。示例默认使用 Python 3.10 及以上版本，主机、账号、密钥和令牌均为实验占位符。
## 一、为什么运维需要 Python
Shell 擅长把现成命令快速串起来，Python 擅长处理复杂逻辑、结构化数据和可维护工程。二者不是替代关系，而是分工关系。
| 场景 | 更适合 Shell | 更适合 Python |
| --- | --- | --- |
| 临时查看进程、磁盘、端口 | 是，命令短且直接 | 通常没有必要 |
| 把两三个系统命令串联 | 是，管道表达清晰 | 流程扩大后再迁移 |
| 解析复杂日志和配置 | 容易出现多层管道 | 是，便于校验和测试 |
| 调用多个接口并处理 JSON | 工具依赖多，错误处理弱 | 是，数据结构天然匹配 |
| 批量 SSH 与结果汇总 | 简单循环尚可 | 是，便于并发、超时和报告 |
| 长期维护的自动化工具 | 容易随需求增长失控 | 是，可模块化和测试 |
| 系统启动早期的小脚本 | 是，依赖少 | 需要确认解释器和环境 |
| 跨平台工具 | 兼容成本较高 | 是，标准库支持更统一 |
可以使用下面的判断方法：
- 一次性、少于十行、主要调用系统命令：优先 Shell。
- 需要解析 JSON、YAML、CSV，或存在多层条件：优先 Python。
- 需要重试、超时、并发、测试、日志、配置管理：优先 Python。
- 已经出现难以理解的 `awk | sed | grep` 长管道：考虑迁移到 Python。
- Python 仍可调用成熟系统命令，不必重新实现 `df`、`ip`、`systemctl` 等能力。
运维开发的重点不只是“脚本能运行”，还包括：失败可观察、执行可重复、配置可管理、输入可校验、代码可测试、工具可分发。
## 二、环境与依赖管理
### 1. 使用虚拟环境
每个工具或项目应拥有独立环境，避免不同项目的依赖版本互相影响。
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install requests paramiko PyYAML
```
Windows 实验环境可执行：
```powershell
py -3.12 -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
```
退出环境：
```bash
deactivate
```
建议始终使用 `python -m pip`，这样可以明确 `pip` 属于当前解释器，减少装错环境的问题。
### 2. 维护依赖清单
简单脚本项目可以使用 `requirements.txt`：
```text
requests==2.32.4
paramiko==3.5.1
PyYAML==6.0.2
```
安装依赖：
```bash
python -m pip install -r requirements.txt
```
生产项目应固定经过验证的版本，并通过依赖更新工具定期升级，而不是永久不更新。开发依赖可以单独放入 `requirements-dev.txt`。
### 3. uv 简介
`uv` 是速度较快的 Python 项目和依赖管理工具，可以创建环境、安装依赖、生成锁文件并运行命令。
```bash
uv init ops-tool
cd ops-tool
uv add requests paramiko pyyaml
uv add --dev pytest ruff
uv run python main.py
```
已有依赖清单时也可以执行：
```bash
uv venv
uv pip install -r requirements.txt
```
学习阶段先理解 `venv` 和 `pip` 的工作方式，再使用 `uv` 提升效率，会更容易排查环境问题。
### 4. 不要向系统 Python 直接装包
很多 Linux 发行版使用系统 Python 支撑包管理器、桌面组件和系统工具。直接执行带管理员权限的 `pip install` 可能造成：
- 覆盖发行版维护的依赖版本。
- 让系统工具因版本不兼容而失效。
- 无法区分哪个项目需要某个包。
- 升级和回滚困难。
- 服务器之间难以复现相同环境。
推荐顺序是：项目虚拟环境、`pipx` 安装独立命令行工具、容器隔离；只有发行版明确要求时才使用系统包管理器安装 Python 包。
## 三、文件与目录操作
### 1. 使用 pathlib 处理路径
`pathlib.Path` 比字符串拼接和大量 `os.path` 调用更直观，也能减少路径分隔符问题。
```python
from pathlib import Path
base_dir = Path("/var/log/myapp")
log_file = base_dir / "service.log"
base_dir.mkdir(parents=True, exist_ok=True)
if log_file.exists() and log_file.is_file():
    size_mb = log_file.stat().st_size / 1024 / 1024
    print(f"日志大小：{size_mb:.2f} 兆字节")
content = log_file.read_text(encoding="utf-8", errors="replace")
```
写配置时可先写临时文件，再原子替换，降低进程读取到半份文件的概率：
```python
from pathlib import Path
config_path = Path("/etc/mytool/config.json")
temp_path = config_path.with_suffix(".tmp")
temp_path.write_text('{"enabled": true}\n', encoding="utf-8")
temp_path.replace(config_path)
```
### 2. 使用 glob 查找文件
```python
from pathlib import Path
log_dir = Path("/var/log")
for log_path in sorted(log_dir.glob("*.log")):
    print(log_path)
for archive in log_dir.rglob("*.log.gz"):
    print(archive)
```
路径模式来自外部输入时，要限制搜索根目录，避免脚本意外扫描整个文件系统。
### 3. 使用 shutil 完成复制和归档
```python
from pathlib import Path
import shutil
source = Path("/etc/myapp/config.yaml")
backup_dir = Path("/var/backups/myapp")
backup_dir.mkdir(parents=True, exist_ok=True)
backup_file = backup_dir / "config.yaml.bak"
shutil.copy2(source, backup_file)
shutil.make_archive(
    base_name=str(backup_dir / "logs"),
    format="gztar",
    root_dir="/var/log/myapp",
)
```
`copy2` 会尽量保留时间等元数据。删除目录前必须校验路径，避免变量为空或指向根目录。
### 4. 迭代读取大文件
不要对数百兆日志直接使用 `read_text()`。文件对象本身就是迭代器，可以逐行处理，内存占用较稳定。
```python
from pathlib import Path
log_path = Path("/var/log/myapp/access.log")
error_count = 0
with log_path.open("r", encoding="utf-8", errors="replace") as file:
    for line_number, line in enumerate(file, start=1):
        if " ERROR " in line:
            error_count += 1
            print(f"第 {line_number} 行：{line.rstrip()}")
print(f"错误总数：{error_count}")
```
处理持续增长的日志时，还要考虑日志轮转、文件被截断和脚本重启后的读取位置。
## 四、子进程与命令执行
### 1. subprocess.run 的推荐写法
```python
import subprocess
result = subprocess.run(
    ["systemctl", "is-active", "nginx"],
    capture_output=True,
    text=True,
    check=True,
    timeout=10,
)
print(result.stdout.strip())
```
关键参数：
- 命令使用字符串列表，每个参数单独一个元素。
- `capture_output=True` 捕获标准输出和标准错误。
- `text=True` 让结果直接返回字符串。
- `check=True` 让非零退出码触发异常。
- `timeout` 防止命令永久阻塞。
需要区分失败类型时：
```python
import subprocess
try:
    result = subprocess.run(
        ["ping", "-c", "1", "192.168.10.20"],
        capture_output=True,
        text=True,
        check=True,
        timeout=5,
    )
except subprocess.TimeoutExpired:
    print("命令执行超时")
except subprocess.CalledProcessError as error:
    print(f"命令失败，退出码：{error.returncode}")
    print(error.stderr.strip())
else:
    print(result.stdout.strip())
```
### 2. 为什么避免 shell=True
`shell=True` 会先让命令经过 Shell 解释。如果命令中拼接了主机名、文件名、分支名等外部输入，攻击者可能注入额外命令。
错误示例：
```python
# 不要把未经校验的输入拼接到 Shell 命令中
subprocess.run(f"ping -c 1 {host}", shell=True, check=True)
```
推荐写法：
```python
subprocess.run(
    ["ping", "-c", "1", host],
    check=True,
    timeout=5,
)
```
只有确实需要管道、重定向或通配符，并且命令内容完全由程序控制时，才考虑 `shell=True`。多数管道任务也可以在 Python 内处理，或用多个 `Popen` 明确连接。
### 3. 与 os.system 的对比
`os.system()` 只能返回退出状态，默认不方便捕获输出、设置超时或得到结构化异常；同时通常依赖 Shell。新代码应优先使用 `subprocess.run()`。
## 五、文本与数据格式处理
### 1. 正则表达式解析日志
常用模式包括：`
- `\d+`：一个或多个数字。
- `\S+`：一个或多个非空白字符。
- `(?P<name>...)`：命名分组，便于读取字段。
- `^` 和 `$`：限制行首与行尾。
- `re.IGNORECASE`：忽略大小写。
下面从访问日志中提取地址、状态码和响应字节数：
```python
import re
pattern = re.compile(
    r'^(?P<ip>\S+) \S+ \S+ \[[^]]+] "[A-Z]+ \S+ [^"]+" '
    r'(?P<status>\d{3}) (?P<size>\d+|-)$'
)
line = '192.168.10.8 - - [12/Aug/2026:10:00:00 +0800] "GET /health HTTP/1.1" 200 18'
match = pattern.match(line)
if match:
    status = int(match.group("status"))
    size = 0 if match.group("size") == "-" else int(match.group("size"))
    print(match.group("ip"), status, size)
```
正则适合稳定格式，不适合替代专用解析器。模式应预编译，并用真实样本和异常样本测试。
### 2. JSON 接口数据
```python
import json
from pathlib import Path
config_path = Path("config.json")
config = json.loads(config_path.read_text(encoding="utf-8"))
print(config["service_name"])
print(json.dumps(config, ensure_ascii=False, indent=2))
```
接口响应通常使用 `response.json()`，但仍需处理响应不是合法 JSON 的情况。
### 3. YAML 配置
安装依赖：
```bash
python -m pip install PyYAML
```
配置示例：
```yaml
service: api-gateway
hosts:
  - 192.168.10.21
  - 192.168.10.22
threshold: 85
```
安全读取：
```python
from pathlib import Path
import yaml
config_path = Path("config.yaml")
with config_path.open("r", encoding="utf-8") as file:
    config = yaml.safe_load(file)
hosts = config.get("hosts", [])
threshold = int(config.get("threshold", 85))
print(hosts, threshold)
```
读取不可信 YAML 时必须使用 `safe_load`，不要使用可构造任意 Python 对象的危险加载方式。读取后还要验证必填字段和数据类型。
### 4. CSV 报表
```python
import csv
from pathlib import Path
report_path = Path("disk-report.csv")
rows = [
    {"host": "192.168.10.21", "usage": 72, "status": "正常"},
    {"host": "192.168.10.22", "usage": 91, "status": "告警"},
]
with report_path.open("w", encoding="utf-8-sig", newline="") as file:
    writer = csv.DictWriter(file, fieldnames=["host", "usage", "status"])
    writer.writeheader()
    writer.writerows(rows)
```
`newline=""` 可避免部分平台出现空行；`utf-8-sig` 便于常见表格软件识别中文。
### 5. INI 配置与 configparser
```ini
[api]
base_url = https://ops.example.invalid
connect_timeout = 3
read_timeout = 10
[alert]
enabled = true
```
```python
from configparser import ConfigParser
from pathlib import Path
parser = ConfigParser()
parser.read(Path("settings.ini"), encoding="utf-8")
base_url = parser.get("api", "base_url")
connect_timeout = parser.getfloat("api", "connect_timeout", fallback=3.0)
alert_enabled = parser.getboolean("alert", "enabled", fallback=False)
print(base_url, connect_timeout, alert_enabled)
```
INI 适合层级简单的配置；复杂嵌套结构可选 YAML。密码和令牌应来自环境变量或密钥管理系统，不应直接写入仓库配置。
## 六、日志记录
`print` 没有级别、时间、模块名和统一输出位置，也不便于日志轮转。长期运行或定时执行的脚本应使用 `logging`。
### 1. 基本配置
```python
import logging
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(levelname)s %(name)s %(message)s",
)
logger = logging.getLogger("disk-check")
logger.info("开始巡检")
logger.warning("磁盘使用率超过阈值：%s%%", 88)
```
日志参数推荐使用延迟格式化，而不是提前拼接字符串，这样被日志级别过滤时可以减少无用格式化。
### 2. 按大小轮转文件
```python
import logging
from logging.handlers import RotatingFileHandler
from pathlib import Path
log_dir = Path("/var/log/ops-tool")
log_dir.mkdir(parents=True, exist_ok=True)
handler = RotatingFileHandler(
    log_dir / "ops-tool.log",
    maxBytes=10 * 1024 * 1024,
    backupCount=5,
    encoding="utf-8",
)
handler.setFormatter(
    logging.Formatter("%(asctime)s %(levelname)s %(name)s %(message)s")
)
logger = logging.getLogger("ops-tool")
logger.setLevel(logging.INFO)
logger.addHandler(handler)
logger.info("服务启动")
```
容器内程序通常把日志写到标准输出，由容器平台收集；传统服务器脚本可以写文件并配置轮转。日志中不要记录密码、令牌、私钥和完整敏感响应。
## 七、错误处理与健壮性
### 1. 捕获明确异常
不要使用空的 `except:` 吞掉所有错误。应捕获能够处理的异常，让未知问题保留堆栈。
```python
import json
from pathlib import Path
config_path = Path("config.json")
try:
    config = json.loads(config_path.read_text(encoding="utf-8"))
except FileNotFoundError:
    print(f"配置文件不存在：{config_path}")
except PermissionError:
    print(f"没有读取权限：{config_path}")
except json.JSONDecodeError as error:
    print(f"配置格式错误，第 {error.lineno} 行第 {error.colno} 列")
else:
    print(f"成功读取配置：{config.get('service', '未命名')}")
```
### 2. 使用 finally 清理资源
文件和网络连接优先使用上下文管理器。无法使用时，可在 `finally` 中确保清理。
```python
lock_file = open("/tmp/mytool.lock", "w", encoding="utf-8")
try:
    print("执行任务")
finally:
    lock_file.close()
```
### 3. 使用退出码表达结果
约定通常是：`0` 表示成功，非零表示失败。监控系统、cron 和 systemd 都会利用退出码判断状态。
```python
import sys

def main() -> int:
    configuration_ok = False
    if not configuration_ok:
        print("配置校验失败", file=sys.stderr)
        return 2
    return 0

if __name__ == "__main__":
    sys.exit(main())
```
可以为参数错误、外部依赖失败和部分巡检失败定义不同退出码，并在工具帮助中说明。
## 八、网络与 API 调用
### 1. requests 的 GET 与 POST
```python
import requests
response = requests.get(
    "https://ops.example.invalid/api/v1/services",
    params={"environment": "test"},
    headers={"Accept": "application/json"},
    timeout=(3.05, 10),
)
response.raise_for_status()
services = response.json()
print(services)
create_response = requests.post(
    "https://ops.example.invalid/api/v1/tasks",
    json={"action": "restart", "host": "192.168.10.21"},
    timeout=(3.05, 10),
)
create_response.raise_for_status()
```
超时必须设置。二元组分别表示连接超时和读取超时。没有超时的请求可能永久占住工作进程。
### 2. 使用 Session 和重试
```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

def build_session() -> requests.Session:
    retry = Retry(
        total=3,
        connect=3,
        read=3,
        backoff_factor=0.5,
        status_forcelist=(429, 500, 502, 503, 504),
        allowed_methods=frozenset({"GET", "HEAD", "OPTIONS"}),
    )
    adapter = HTTPAdapter(max_retries=retry)
    session = requests.Session()
    session.mount("http://", adapter)
    session.mount("https://", adapter)
    session.headers.update({"User-Agent": "ops-health-check/1.0"})
    return session
```
默认只重试幂等请求。对创建资源的 POST 请求盲目重试，可能产生重复任务；需要接口提供幂等键或由调用方确认语义。
### 3. 查询服务健康接口
```python
import sys
import requests

def check_health(url: str) -> bool:
    try:
        response = requests.get(url, timeout=(2, 5))
        response.raise_for_status()
        data = response.json()
    except (requests.RequestException, ValueError) as error:
        print(f"健康检查失败：{error}", file=sys.stderr)
        return False
    return data.get("status") == "ok"

if __name__ == "__main__":
    target = "http://192.168.10.30:8080/health"
    sys.exit(0 if check_health(target) else 1)
```
生产环境还要考虑证书校验、代理、认证头、响应结构校验和敏感信息脱敏。不要为了省事设置 `verify=False`。
## 九、SSH 自动化
### 1. 使用 Paramiko 执行远程命令
安装：
```bash
python -m pip install paramiko
```
示例使用密钥认证，并拒绝未知主机密钥：
```python
from pathlib import Path
import paramiko
host = "192.168.10.21"
username = "opsuser"
key_path = Path.home() / ".ssh" / "id_ed25519"
client = paramiko.SSHClient()
client.load_system_host_keys()
client.set_missing_host_key_policy(paramiko.RejectPolicy())
try:
    client.connect(
        hostname=host,
        username=username,
        key_filename=str(key_path),
        timeout=5,
        banner_timeout=5,
        auth_timeout=5,
    )
    _, stdout, stderr = client.exec_command("uptime", timeout=10)
    exit_code = stdout.channel.recv_exit_status()
    output = stdout.read().decode("utf-8", errors="replace").strip()
    error_output = stderr.read().decode("utf-8", errors="replace").strip()
    if exit_code != 0:
        raise RuntimeError(f"远程命令失败：{error_output}")
    print(output)
finally:
    client.close()
```
首次连接前应通过可信渠道获取主机公钥并写入 `known_hosts`，不要在生产代码中无条件接受未知密钥。
### 2. 密钥认证建议
- 为自动化任务创建权限最小化的独立账号。
- 私钥设置严格文件权限，并使用密钥托管或安全注入。
- 远程 `sudo` 只放行所需命令，不授予无限权限。
- 定期轮换密钥，清理离职人员和废弃任务的授权。
- 对目标主机清单和命令参数进行校验与审计。
### 3. Fabric 简介
Fabric 基于 Paramiko 提供更高层的连接和任务接口，适合编排少量服务器操作：
```python
from fabric import Connection
with Connection(
    host="192.168.10.21",
    user="opsuser",
    connect_kwargs={"key_filename": "~/.ssh/id_ed25519"},
) as connection:
    result = connection.run("uname -a", hide=True)
    print(result.stdout.strip())
```
主机规模和状态管理需求扩大后，应优先考虑 Ansible 等声明式工具，而不是自行维护复杂 SSH 编排平台。
## 十、定时与调度
### 1. 使用 cron 调用虚拟环境
cron 环境变量很少，必须使用绝对路径，并明确工作目录和日志位置：
```cron
*/5 * * * * cd /opt/ops-tool && /opt/ops-tool/.venv/bin/python check.py >> /var/log/ops-tool/check.log 2>&1
```
脚本应返回可靠退出码。敏感环境变量不要直接写在所有用户可读的 crontab 中。
### 2. 使用 systemd timer
服务单元 `/etc/systemd/system/disk-check.service`：
```ini
[Unit]
Description=运行磁盘巡检脚本
[Service]
Type=oneshot
User=ops
WorkingDirectory=/opt/disk-check
ExecStart=/opt/disk-check/.venv/bin/python /opt/disk-check/check.py
```
定时器单元 `/etc/systemd/system/disk-check.timer`：
```ini
[Unit]
Description=每五分钟运行磁盘巡检
[Timer]
OnBootSec=2min
OnUnitActiveSec=5min
Persistent=true
[Install]
WantedBy=timers.target
```
启用并查看：
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now disk-check.timer
systemctl list-timers disk-check.timer
journalctl -u disk-check.service
```
与 cron 相比，systemd timer 更方便查看状态、日志、资源限制和依赖关系。
### 3. 脚本内使用 flock 防止重复运行
下面模式仅适用于支持 `fcntl` 的 Linux 和类 Unix 系统：
```python
from pathlib import Path
import fcntl
import os
import sys

class ProcessLock:
    def __init__(self, path: Path) -> None:
        self.path = path
        self.file = None
    def __enter__(self) -> "ProcessLock":
        self.path.parent.mkdir(parents=True, exist_ok=True)
        self.file = self.path.open("w", encoding="utf-8")
        try:
            fcntl.flock(self.file.fileno(), fcntl.LOCK_EX | fcntl.LOCK_NB)
        except BlockingIOError:
            self.file.close()
            raise RuntimeError("已有任务正在运行") from None
        self.file.write(str(os.getpid()))
        self.file.flush()
        return self
    def __exit__(self, exc_type, exc_value, traceback) -> None:
        if self.file is not None:
            fcntl.flock(self.file.fileno(), fcntl.LOCK_UN)
            self.file.close()

if __name__ == "__main__":
    try:
        with ProcessLock(Path("/run/user/1000/ops-tool.lock")):
            print("执行定时任务")
    except RuntimeError as error:
        print(error, file=sys.stderr)
        sys.exit(1)
```
锁文件内容不是锁本身，真正的互斥由打开文件描述符上的 `flock` 保持。任务异常退出后，内核会释放锁。
## 十一、实战脚本示例
### 1. 批量服务器磁盘巡检并输出表格报告
依赖 `paramiko`。主机文件每行一个地址，空行和以 `#` 开头的行会被忽略。脚本使用 SSH 密钥并输出超过阈值的文件系统。
```python
#!/usr/bin/env python3
import argparse
from concurrent.futures import ThreadPoolExecutor, as_completed
from dataclasses import dataclass
from pathlib import Path
import sys
import paramiko

@dataclass(frozen=True)
class DiskRecord:
    host: str
    filesystem: str
    mountpoint: str
    usage: int


def load_hosts(path: Path) -> list[str]:
    hosts = [
        line.strip()
        for line in path.read_text(encoding="utf-8").splitlines()
        if line.strip() and not line.lstrip().startswith("#")
    ]
    if not hosts:
        raise ValueError("主机清单为空")
    return hosts


def parse_df(host: str, text: str) -> list[DiskRecord]:
    records = []
    for line in text.splitlines()[1:]:
        fields = line.split()
        if len(fields) >= 6 and fields[4].endswith("%"):
            records.append(
                DiskRecord(host, fields[0], fields[5], int(fields[4][:-1]))
            )
    return records


def inspect(host: str, user: str, key: Path) -> list[DiskRecord]:
    client = paramiko.SSHClient()
    client.load_system_host_keys()
    client.set_missing_host_key_policy(paramiko.RejectPolicy())
    try:
        client.connect(
            host,
            username=user,
            key_filename=str(key),
            timeout=5,
            banner_timeout=5,
            auth_timeout=5,
        )
        _, stdout, stderr = client.exec_command(
            "df -P -x tmpfs -x devtmpfs", timeout=15
        )
        code = stdout.channel.recv_exit_status()
        output = stdout.read().decode("utf-8", errors="replace")
        error = stderr.read().decode("utf-8", errors="replace").strip()
        if code:
            raise RuntimeError(error or f"远程退出码为 {code}")
        return parse_df(host, output)
    finally:
        client.close()


def print_report(records: list[DiskRecord], threshold: int) -> None:
    print(f"{'主机':<16} {'文件系统':<20} {'挂载点':<18} {'使用率':>7}  状态")
    print("-" * 76)
    for item in sorted(records, key=lambda value: (value.host, -value.usage)):
        status = "告警" if item.usage >= threshold else "正常"
        print(
            f"{item.host:<16} {item.filesystem:<20} {item.mountpoint:<18} "
            f"{item.usage:>6}%  {status}"
        )


def parse_args() -> argparse.Namespace:
    parser = argparse.ArgumentParser(description="批量检查服务器磁盘使用率")
    parser.add_argument("--hosts", required=True, type=Path, help="主机清单")
    parser.add_argument("--user", default="opsuser", help="SSH 用户")
    parser.add_argument(
        "--key", type=Path, default=Path.home() / ".ssh/id_ed25519"
    )
    parser.add_argument("--threshold", type=int, default=85)
    parser.add_argument("--workers", type=int, default=8)
    return parser.parse_args()


def main() -> int:
    args = parse_args()
    if not 1 <= args.threshold <= 100 or args.workers < 1:
        print("阈值或并发数无效", file=sys.stderr)
        return 2
    try:
        hosts = load_hosts(args.hosts)
    except (OSError, ValueError) as error:
        print(f"读取主机清单失败：{error}", file=sys.stderr)
        return 2

    records: list[DiskRecord] = []
    failures = 0
    with ThreadPoolExecutor(max_workers=args.workers) as executor:
        tasks = {
            executor.submit(inspect, host, args.user, args.key.expanduser()): host
            for host in hosts
        }
        for task in as_completed(tasks):
            try:
                records.extend(task.result())
            except Exception as error:
                failures += 1
                print(f"{tasks[task]} 巡检失败：{error}", file=sys.stderr)

    print_report(records, args.threshold)
    alerts = sum(item.usage >= args.threshold for item in records)
    return 1 if failures or alerts else 0


if __name__ == "__main__":
    sys.exit(main())
```
运行方式：
```bash
python disk_check.py --hosts hosts.txt --user opsuser --threshold 85
```
### 2. 日志关键字统计分析
该脚本逐行读取一个或多个日志文件，按关键字统计次数，并列出首次和最后一次匹配行号。
```python
#!/usr/bin/env python3
import argparse
from pathlib import Path
import re
import sys


def analyze(path: Path, patterns: dict[str, re.Pattern[str]]) -> dict[str, int]:
    counts = dict.fromkeys(patterns, 0)
    with path.open("r", encoding="utf-8", errors="replace") as file:
        for line in file:
            for name, pattern in patterns.items():
                if pattern.search(line):
                    counts[name] += 1
    return counts


def parse_args() -> argparse.Namespace:
    parser = argparse.ArgumentParser(description="统计日志中的关键字")
    parser.add_argument("files", nargs="+", type=Path, help="日志文件")
    parser.add_argument(
        "--keyword", action="append", required=True, help="可重复指定正则"
    )
    parser.add_argument("--ignore-case", action="store_true")
    parser.add_argument("--literal", action="store_true", help="按普通文本匹配")
    return parser.parse_args()


def main() -> int:
    args = parse_args()
    flags = re.IGNORECASE if args.ignore_case else 0
    try:
        patterns = {
            word: re.compile(re.escape(word) if args.literal else word, flags)
            for word in args.keyword
        }
    except re.error as error:
        print(f"正则表达式错误：{error}", file=sys.stderr)
        return 2

    failed = False
    for path in args.files:
        try:
            counts = analyze(path, patterns)
        except OSError as error:
            print(f"{path} 读取失败：{error}", file=sys.stderr)
            failed = True
            continue
        print(f"\n文件：{path}")
        print(f"{'关键字':<30} {'次数':>8}")
        print("-" * 40)
        for keyword, count in counts.items():
            print(f"{keyword:<30} {count:>8}")
    return 1 if failed else 0


if __name__ == "__main__":
    sys.exit(main())
```
运行方式：
```bash
python log_stat.py /var/log/myapp/*.log --keyword ERROR --keyword 'timeout|超时' --ignore-case
```
### 3. Webhook 告警通知封装
该脚本从环境变量读取 Webhook 地址，支持重试、超时和命令行调用。示例负载是通用 JSON，实际使用时按告警平台接口调整字段。
```python
#!/usr/bin/env python3
import argparse
from dataclasses import asdict, dataclass
import os
import sys
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

@dataclass(frozen=True)
class Alert:
    title: str
    message: str
    severity: str = "警告"
    source: str = "ops-script"


def build_session() -> requests.Session:
    retry = Retry(
        total=3,
        backoff_factor=0.5,
        status_forcelist=(429, 500, 502, 503, 504),
        allowed_methods=frozenset({"POST"}),
    )
    session = requests.Session()
    adapter = HTTPAdapter(max_retries=retry)
    session.mount("http://", adapter)
    session.mount("https://", adapter)
    return session


def send_alert(url: str, alert: Alert) -> None:
    with build_session() as session:
        response = session.post(url, json=asdict(alert), timeout=(3.05, 10))
        response.raise_for_status()


def parse_args() -> argparse.Namespace:
    parser = argparse.ArgumentParser(description="发送 Webhook 告警")
    parser.add_argument("--title", required=True)
    parser.add_argument("--message", required=True)
    parser.add_argument("--severity", choices=("信息", "警告", "严重"), default="警告")
    parser.add_argument("--source", default="ops-script")
    return parser.parse_args()


def main() -> int:
    args = parse_args()
    url = os.environ.get("OPS_WEBHOOK_URL")
    if not url:
        print("缺少环境变量 OPS_WEBHOOK_URL", file=sys.stderr)
        return 2
    try:
        send_alert(url, Alert(args.title, args.message, args.severity, args.source))
    except requests.RequestException as error:
        print(f"告警发送失败：{error}", file=sys.stderr)
        return 1
    print("告警发送成功")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```
运行方式：
```bash
export OPS_WEBHOOK_URL='https://alert.example.invalid/hooks/占位令牌'
python webhook_alert.py --title '磁盘告警' --message '192.168.10.21 根分区使用率达到 92%' --severity 严重
```
Webhook 重试 POST 的前提是接收端能容忍重复消息，最好由负载携带唯一事件编号并由服务端去重。
## 十二、打包与分发
### 1. 使用 argparse 制作 CLI
下面是一个完整的服务状态检查命令，可通过参数指定地址、期望状态和超时：
```python
#!/usr/bin/env python3
import argparse
import sys
import requests

def parse_args() -> argparse.Namespace:
    parser = argparse.ArgumentParser(
        prog="service-check",
        description="检查 HTTP 服务健康状态",
    )
    parser.add_argument("url", help="健康检查地址")
    parser.add_argument(
        "--expected",
        default="ok",
        help="期望的 status 字段值",
    )
    parser.add_argument(
        "--timeout",
        type=float,
        default=5.0,
        help="请求超时秒数",
    )
    parser.add_argument("--verbose", action="store_true", help="显示响应内容")
    return parser.parse_args()

def main() -> int:
    args = parse_args()
    if args.timeout <= 0:
        print("超时必须大于零", file=sys.stderr)
        return 2
    try:
        response = requests.get(args.url, timeout=(3.05, args.timeout))
        response.raise_for_status()
        data = response.json()
    except (requests.RequestException, ValueError) as error:
        print(f"检查失败：{error}", file=sys.stderr)
        return 1
    if args.verbose:
        print(data)
    actual = data.get("status")
    if actual != args.expected:
        print(f"状态异常，期望 {args.expected}，实际 {actual}", file=sys.stderr)
        return 1
    print("服务状态正常")
    return 0

if __name__ == "__main__":
    sys.exit(main())
```
### 2. 使用 pyproject.toml 声明入口点
推荐目录：
```text
service-check/
├── pyproject.toml
├── src/
│   └── service_check/
│       ├── __init__.py
│       └── cli.py
└── tests/
    └── test_cli.py
```
`pyproject.toml` 最小示例：
```toml
[build-system]
requires = ["setuptools>=75"]
build-backend = "setuptools.build_meta"
[project]
name = "service-check"
version = "0.1.0"
description = "内部服务健康检查工具"
requires-python = ">=3.10"
dependencies = ["requests>=2.32,<3"]
[project.scripts]
service-check = "service_check.cli:main"
```
开发安装后可直接执行入口命令：
```bash
python -m pip install -e .
service-check http://192.168.10.30:8080/health
```
入口点让调用方不再关心脚本文件路径，也便于版本化、构建软件包和内部仓库分发。
## 十三、代码质量与测试
### 1. Ruff 与 Black
Ruff 可执行静态检查和导入排序，Black 负责统一格式。团队也可以只选 Ruff 的格式化器，关键是统一配置并放入持续集成。
```bash
python -m pip install ruff black pytest
ruff check .
ruff check . --fix
black .
```
`pyproject.toml` 示例：
```toml
[tool.ruff]
line-length = 88
target-version = "py310"
[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP"]
[tool.black]
line-length = 88
target-version = ["py310"]
```
### 2. 类型注解入门
类型注解帮助编辑器和检查工具提前发现参数、返回值和空值处理问题，但不会自动替代运行时校验。
```python
from pathlib import Path

def load_targets(path: Path) -> list[str]:
    return [
        line.strip()
        for line in path.read_text(encoding="utf-8").splitlines()
        if line.strip() and not line.startswith("#")
    ]

def usage_status(usage: int, threshold: int = 85) -> str:
    if not 0 <= usage <= 100:
        raise ValueError("使用率必须在 0 到 100 之间")
    return "告警" if usage >= threshold else "正常"
```
可进一步使用 Pyright 或 mypy 做静态类型检查。
### 3. pytest 最小测试
假设 `disk.py` 中包含：
```python
def usage_status(usage: int, threshold: int = 85) -> str:
    if not 0 <= usage <= 100:
        raise ValueError("使用率必须在 0 到 100 之间")
    return "告警" if usage >= threshold else "正常"
```
测试文件 `test_disk.py`：
```python
import pytest
from disk import usage_status

def test_usage_below_threshold_is_normal() -> None:
    assert usage_status(84) == "正常"

def test_usage_at_threshold_is_alert() -> None:
    assert usage_status(85) == "告警"

def test_invalid_usage_raises_error() -> None:
    with pytest.raises(ValueError, match="0 到 100"):
        usage_status(101)
```
运行：
```bash
pytest -q
```
运维脚本优先测试纯函数，例如配置校验、日志解析、阈值判断和命令输出解析。网络、文件和 SSH 可通过夹具与模拟对象隔离，避免测试依赖真实生产环境。
## 十四、进阶方向
### 1. 使用 FastAPI 编写内部运维接口
```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
app = FastAPI(title="内部运维接口")

class RestartRequest(BaseModel):
    service: str
    host: str

@app.get("/health")
def health() -> dict[str, str]:
    return {"status": "ok"}

@app.post("/restart")
def restart(request: RestartRequest) -> dict[str, str]:
    allowed_hosts = {"192.168.10.21", "192.168.10.22"}
    if request.host not in allowed_hosts:
        raise HTTPException(status_code=403, detail="主机不在允许清单中")
    return {"status": "accepted", "service": request.service, "host": request.host}
```
启动实验服务：
```bash
python -m pip install fastapi uvicorn
uvicorn app:app --host 127.0.0.1 --port 8000
```
真实运维接口必须增加身份认证、授权、审计、幂等控制、输入校验、任务队列和操作审批，不能直接把任意系统命令暴露为接口。
### 2. Ansible 模块开发概念
当现有模块不能满足资源管理需求时，可以编写自定义模块。模块接收参数、检查当前状态、执行最小变更，并返回 `changed`、结果和错误信息。重点是幂等性：重复执行不应产生额外副作用。学习时可先编写只读事实采集模块，再尝试支持检查模式的资源模块。
### 3. 云平台 SDK
云厂商和基础设施平台通常提供 Python SDK，例如 AWS 的 `boto3`。SDK 比手工拼接 HTTP 请求更方便处理签名、分页、区域和错误类型。常见实践包括：
- 分页遍历实例和磁盘资源。
- 按标签筛选资源。
- 自动生成资产清单。
- 检查安全组、快照和闲置资源。
- 使用角色或临时凭据，避免长期密钥写入文件。
批量修改云资源前应先提供只读预览、变更数量上限和明确确认机制。
## 十五、学习路线与实践任务
建议按“文件与命令 → 数据与接口 → SSH 与调度 → 工具化与测试 → 平台化”的顺序推进。
- [ ] 使用 `pathlib` 编写日志归档工具，支持保留天数和演练模式。
- [ ] 使用 `subprocess.run` 编写服务状态检查，覆盖超时和非零退出码。
- [ ] 解析一份真实脱敏访问日志，输出状态码和来源地址排行。
- [ ] 读取 YAML 主机清单，并验证地址、端口和阈值字段。
- [ ] 为脚本加入标准日志、明确退出码和错误分类。
- [ ] 调用实验服务健康接口，加入超时、重试和响应校验。
- [ ] 使用 Paramiko 巡检两台 `192.168.x.x` 实验主机。
- [ ] 将巡检结果写入 CSV，并对失败主机单独汇总。
- [ ] 使用 cron 或 systemd timer 定时运行脚本。
- [ ] 使用 `flock` 防止慢任务重叠执行。
- [ ] 把 Webhook 地址从代码迁移到环境变量或密钥服务。
- [ ] 使用 `argparse` 为脚本补齐帮助、参数类型和默认值。
- [ ] 使用 `pyproject.toml` 把脚本安装为命令行工具。
- [ ] 使用 Ruff 和 Black 统一代码质量检查。
- [ ] 为日志解析和阈值判断编写至少五个 pytest 用例。
- [ ] 在持续集成中执行格式检查、静态检查和测试。
- [ ] 使用 FastAPI 包装一个只读巡检接口，并加入简单认证。
- [ ] 阅读一个 Ansible 内置模块，理解参数、幂等和返回结构。
- [ ] 使用云 SDK 生成只读资产报告，不执行修改操作。
- [ ] 为最终工具补充依赖锁定、示例配置、运行手册和回滚方式。
每个练习都应至少回答以下问题：
1. 输入是否经过校验？
2. 外部操作是否有超时？
3. 失败是否有日志和非零退出码？
4. 重试会不会产生重复副作用？
5. 凭据是否脱离代码和仓库？
6. 核心逻辑是否能在无生产依赖的环境中测试？
## 十六、总结
Python 在运维中的价值，是把临时命令提升为可维护、可测试、可审计的自动化工具。Shell 继续承担短小直接的系统编排，Python 负责复杂逻辑、结构化数据、网络接口、并发巡检和工具工程化。
实践时应优先养成几项习惯：使用虚拟环境隔离依赖，使用 `pathlib` 和 `subprocess.run` 等现代接口，为所有外部操作设置超时，明确处理异常和退出码，用 `logging` 代替零散输出，把凭据移出代码，并为解析与判断逻辑编写测试。
当单机脚本逐渐具备配置、日志、测试、CLI、打包、接口和权限控制后，就已经从“会写脚本”迈向了“能够交付运维工具”。下一步应结合真实但脱敏的实验环境持续迭代，并逐步学习配置管理、云 SDK、内部平台和 SRE 工程实践。
