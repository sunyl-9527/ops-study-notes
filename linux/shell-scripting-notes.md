# Shell 脚本编程笔记

> 适用对象：进阶自 [linux-shell-command-handbook.md](./linux-shell-command-handbook.md)，面向想把日常操作脚本化的运维学习者。

---
## 一、认识 Shell 脚本
Shell 脚本把已经会手动执行的 Linux 命令组织成可重复、可检查、可定时运行的流程。常见运维用途包括服务巡检、日志清理、备份、批量配置和部署辅助。
本文统一使用 Bash，因为会用到 `[[ ]]`、数组和进程替换等 Bash 特性：
```bash
bash --version
mkdir -p "$HOME/shell-lab"
cd "$HOME/shell-lab" || exit 1
```
学习时先在终端逐条验证命令，再写进脚本；先在虚拟机或测试主机验证，再考虑定时任务和生产环境。
### 1. shebang
脚本第一行指定解释器：
```bash
#!/usr/bin/env bash
```
它从 `PATH` 查找 Bash，适合不同发行版。解释器位置固定时也可使用：
```bash
#!/bin/bash
```
第一个脚本 `hello.sh`：
```bash
#!/usr/bin/env bash

printf '主机：%s\n' "$(hostname)"
printf '时间：%s\n' "$(date '+%F %T')"
printf '用户：%s\n' "$USER"
```
### 2. 执行方式
```bash
chmod +x hello.sh
./hello.sh
bash hello.sh
source hello.sh
```
- `./hello.sh`：在子进程中按 shebang 执行，日常脚本优先使用。
- `bash hello.sh`：明确交给 Bash，忽略 shebang 指定的解释器。
- `source hello.sh`：在当前 Shell 执行，会改变当前变量和目录，只用于加载环境或函数库。
### 3. 退出码
命令成功通常返回 `0`，失败返回非 `0`：
```bash
ls /etc/hosts
printf '退出码：%s\n' "$?"
```
脚本主动返回状态：
```bash
if [[ ! -r /etc/hosts ]]; then
    printf '错误：文件不可读\n' >&2
    exit 1
fi
exit 0
```
`cron`、systemd、监控和 CI/CD 都依赖退出码判断结果。错误信息应写到标准错误，并返回非 `0`。
```bash
mkdir -p backup && printf '创建成功\n'
systemctl is-active --quiet nginx || printf 'nginx 未运行\n' >&2
cd /srv/app || exit 1
```
`&&` 表示成功后继续，`||` 表示失败后处理。涉及 `cd`、复制、删除和重启时必须检查结果。
---
## 二、变量、引用与参数展开
### 1. 定义变量
赋值时等号两边不能有空格：
```bash
service_name="nginx"
retry_count=3
readonly DEFAULT_PORT=8080
export APP_ENV="test"

printf '服务：%s，重试：%s\n' "$service_name" "$retry_count"
```
小写变量通常用于脚本内部，大写变量常用于环境变量或常量。密码和令牌不要写进脚本或提交到 Git。
### 2. 引号规则
```bash
name="ops"
printf '%s\n' '$name'                 # 原样输出 $name
printf '%s\n' "$name"                 # 展开变量
printf '%s\n' "$(date +%F)"           # 执行命令替换
```
不加引号会触发 word splitting 和 glob 展开：
```bash
path="/srv/app data"
printf '%s\n' $path                     # 错误：可能拆成两个参数
printf '%s\n' "$path"                   # 正确
rm -- "$path"                           # -- 防止文件名被当成选项
```
脚本中除非明确需要拆词，否则变量展开一律加双引号。
### 3. `${}` 展开
```bash
name="app"
archive="${name}_$(date +%F).tar.gz"
file="application.log"

printf '%s\n' "${#file}"               # 长度
printf '%s\n' "${file%.*}"             # application
printf '%s\n' "${file##*.}"            # log
printf '%s\n' "${file/log/txt}"        # application.txt
```
`${name}_backup` 能明确变量边界；`$name_backup` 会读取另一个变量。
### 4. 默认值展开
```bash
port="${PORT:-8080}"                    # 空或未设置时读取默认值
: "${LOG_DIR:=/var/log/myapp}"          # 同时给变量赋默认值
: "${BACKUP_DIR:?必须设置 BACKUP_DIR}"  # 缺少必填值时终止
mode="${DEBUG:+debug}"                  # 有值时使用替代内容
```
这类展开常用于让脚本配置可由环境变量覆盖。
### 5. 命令替换与整数运算
```bash
today="$(date +%F)"
retry=0
((retry += 1))
printf '%s %d\n' "$today" "$retry"
```
使用 `$()` 而不是反引号。启用 `set -e` 时避免单独使用首次返回值为 `0` 的 `((retry++))`，可写成 `((retry += 1))`。
---
## 三、条件判断
### 1. `test`、`[ ]` 和 `[[ ]]`
```bash
test -f /etc/hosts
[ -f /etc/hosts ]
[[ -f /etc/hosts ]]
```
`[` 是命令，因此两侧必须留空格。需要兼容 `/bin/sh` 时使用 `[ ]`；明确写 Bash 时优先 `[[ ]]`，它更安全并支持模式和正则。
### 2. 字符串判断
```bash
environment="production"

[[ -z "$environment" ]] && printf '字符串为空\n'
[[ -n "$environment" ]] && printf '字符串非空\n'
[[ "$environment" == "production" ]] && printf '生产环境\n'
[[ "$environment" != "test" ]] && printf '不是测试环境\n'

file="access.log"
[[ "$file" == *.log ]] && printf '日志文件\n'     # 模式不要加引号

port="8080"
if [[ "$port" =~ ^[0-9]+$ ]]; then
    printf '端口格式正确\n'
fi
```
### 3. 数字判断
```bash
used_percent=87

if ((used_percent >= 90)); then
    printf '严重告警\n'
elif ((used_percent >= 80)); then
    printf '普通告警\n'
else
    printf '状态正常\n'
fi
```
在 `[ ]` 中使用 `-eq`、`-ne`、`-lt`、`-le`、`-gt`、`-ge`。Bash 整数条件优先使用 `(( ))`。
### 4. 文件测试
```bash
target="/etc/ssh/sshd_config"

[[ -e "$target" ]] && printf '路径存在\n'
[[ -f "$target" ]] && printf '普通文件\n'
[[ -d /etc/ssh ]] && printf '目录存在\n'
[[ -r "$target" ]] && printf '可读\n'
[[ -w "$target" ]] && printf '可写\n'
[[ -x /usr/bin/ssh ]] && printf '可执行\n'
[[ -s "$target" ]] && printf '文件非空\n'
[[ -L "$target" ]] && printf '符号链接\n'
```
运维中常在修改配置、重启服务和处理备份前检查路径类型及权限。
### 5. 完整分支
```bash
if command -v curl >/dev/null 2>&1; then
    printf 'curl 已安装\n'
elif command -v wget >/dev/null 2>&1; then
    printf '仅检测到 wget\n'
else
    printf '错误：缺少下载工具\n' >&2
    exit 3
fi
```
---
## 四、循环与安全遍历文件
### 1. `for` 循环
```bash
for service in nginx sshd cron; do
    printf '检查：%s\n' "$service"
done

servers=("192.168.56.11" "192.168.56.12")
for server in "${servers[@]}"; do
    printf '主机：%s\n' "$server"
done

for ((attempt = 1; attempt <= 3; attempt++)); do
    printf '第 %d 次尝试\n' "$attempt"
done
```
`for` 适合固定列表、数组、参数列表和有限重试。
### 2. `while` 循环
```bash
while IFS= read -r server || [[ -n "$server" ]]; do
    [[ -z "$server" || "$server" == \#* ]] && continue
    printf '读取主机：%s\n' "$server"
done < servers.txt
```
`IFS=` 保留首尾空白，`read -r` 保留反斜杠，附加条件可处理末行没有换行符的文件。
### 3. `until` 循环
```bash
attempt=1
until curl --fail --silent --max-time 2 \
    "http://192.168.56.20:8080/health" >/dev/null; do
    if ((attempt >= 5)); then
        printf '服务未就绪\n' >&2
        exit 4
    fi
    sleep 2
    ((attempt += 1))
done
```
`until` 适合等待服务、文件或锁，但必须设置次数或超时。`continue` 跳过当前轮，`break` 结束循环。
### 4. 正确遍历文件
非递归 glob：
```bash
shopt -s nullglob
files=(/var/log/myapp/*.log)
for file in "${files[@]}"; do
    printf '处理：%s\n' "$file"
done
```
`nullglob` 避免无匹配时把字面量 `*.log` 当文件名。
递归处理任意文件名：
```bash
count=0
while IFS= read -r -d '' file; do
    printf '发现：%s\n' "$file"
    ((count += 1))
done < <(find /var/log/myapp -type f -name '*.log' -print0)
printf '总数：%d\n' "$count"
```
不要使用 `for file in $(find ...)` 或解析 `ls`，因为空格、换行和通配符会破坏文件名边界。
---
## 五、函数
### 1. 参数与 `local`
```bash
check_file() {
    local file="$1"
    local label="${2:-目标文件}"

    if [[ -r "$file" ]]; then
        printf '%s可读：%s\n' "$label" "$file"
        return 0
    fi

    printf '%s不可读：%s\n' "$label" "$file" >&2
    return 1
}

check_file "/etc/hosts" "主机配置"
```
函数适合封装日志、校验、告警和重复操作。函数变量默认可能影响外部作用域，因此内部变量应使用 `local`。
### 2. 返回状态与输出数据
`return` 只能返回 `0` 到 `255` 的状态码：
```bash
is_service_active() {
    local service="$1"
    systemctl is-active --quiet "$service"
}

if is_service_active nginx; then
    printf 'nginx 正常\n'
fi
```
返回字符串时写标准输出，再用命令替换接收：
```bash
make_archive_name() {
    local prefix="$1"
    printf '%s_%s.tar.gz\n' "$prefix" "$(date '+%Y%m%d_%H%M%S')"
}

archive_name="$(make_archive_name app)"
```
函数中的日志写到标准错误，避免污染命令替换结果。
---
## 六、参数处理
### 1. 特殊参数
```text
$0  脚本名        $1、$2  位置参数
$#  参数数量      $@      所有参数并保留边界
$*  所有参数      $?      上一命令退出码
$$  当前进程号
```
基本校验：
```bash
if (($# != 2)); then
    printf '用法：%s <源目录> <目标目录>\n' "$0" >&2
    exit 2
fi
```
### 2. `"$@"` 与 `"$*"`
```bash
show_args() {
    local arg
    for arg in "$@"; do
        printf '<%s>\n' "$arg"
    done
}
show_args "app server" "8080"
```
转发参数几乎总是使用 `"$@"`；`"$*"` 会把所有参数合成一个字符串，通常只适合日志文本。
### 3. `shift`
```bash
while (($# > 0)); do
    case "$1" in
        --verbose) verbose=true; shift ;;
        --output)
            (($# >= 2)) || { printf '缺少输出路径\n' >&2; exit 2; }
            output="$2"
            shift 2
            ;;
        *) printf '未知参数：%s\n' "$1" >&2; exit 2 ;;
    esac
done
```
`shift` 删除当前 `$1` 并让其余参数前移，适合手动解析长选项。
### 4. `getopts`
```bash
file=""
count=1
verbose=false

usage() { printf '用法：%s -f <文件> [-n 次数] [-v]\n' "$0"; }

while getopts ":f:n:vh" option; do
    case "$option" in
        f) file="$OPTARG" ;;
        n) count="$OPTARG" ;;
        v) verbose=true ;;
        h) usage; exit 0 ;;
        :) printf '选项 -%s 缺少值\n' "$OPTARG" >&2; exit 2 ;;
        \?) printf '未知选项：-%s\n' "$OPTARG" >&2; exit 2 ;;
    esac
done
shift $((OPTIND - 1))

[[ -n "$file" ]] || { usage >&2; exit 2; }
[[ "$count" =~ ^[1-9][0-9]*$ ]] || exit 2
```
`getopts` 适合短选项。复杂长选项可使用 `case` 配合 `shift`。
---
## 七、输入输出与重定向
每个进程默认有标准输入 `0`、标准输出 `1` 和标准错误 `2`。
```bash
printf '正常信息\n'
printf '错误信息\n' >&2
command > output.log
command >> output.log
command 2> error.log
command > output.log 2>&1
command >/dev/null 2>&1
printf '开始任务\n' | tee -a task.log
```
`>` 覆盖，`>>` 追加。正常数据写标准输出，诊断信息写标准错误，便于调用方分别处理。
### 1. 管道
```bash
journalctl -u nginx --since today |
grep -i 'error' |
wc -l
```
管道连接前一命令的标准输出与后一命令的标准输入。启用 `pipefail` 后，任一环节失败都会让管道失败：
```bash
set -o pipefail
false | true
printf '%s\n' "${PIPESTATUS[@]}"
```
### 2. here-document
```bash
port=8080
cat > app.conf <<EOF
环境=测试
端口=$port
EOF
```
引用结束标记可禁止展开：
```bash
cat <<'EOF'
$HOME 和 $(date) 都会保持原样。
EOF
```
远程批量命令：
```bash
ssh "ops@<your-server>" 'bash -s' <<'REMOTE'
set -e
hostname
uptime
systemctl is-active nginx
REMOTE
```
### 3. here-string 与进程替换
```bash
read -r user_name user_role <<< "ops admin"
diff <(sort expected.txt) <(sort actual.txt)
```
here-string 适合向命令传短字符串；进程替换适合把命令输出当临时输入，也是避免 `while` 管道子 Shell 的常见方式。
---
## 八、错误处理与健壮性
### 1. 严格模式
```bash
#!/usr/bin/env bash
set -Eeuo pipefail
```
- `-e`：未处理的命令失败时退出。
- `-E`：函数和命令替换继承 `ERR` trap。
- `-u`：读取未设置变量时报错。
- `pipefail`：管道任一命令失败时返回失败。
严格模式不能代替显式处理。预期可能失败的命令放进条件：
```bash
if ! systemctl is-active --quiet nginx; then
    printf 'nginx 未运行\n' >&2
fi

curl --fail "http://192.168.56.20:8080/health" || {
    printf '接口检查失败\n' >&2
    exit 4
}
```
可选变量在 `set -u` 下使用 `${OPTIONAL_VALUE:-}`，必填变量使用 `${VALUE:?错误信息}`。
### 2. `trap` 与错误位置
```bash
on_error() {
    local code="$1"
    local line="$2"
    printf '第 %s 行失败，退出码 %s\n' "$line" "$code" >&2
}
trap 'on_error "$?" "$LINENO"' ERR
```
`trap` 常用于记录错误，以及在 `EXIT`、`INT`、`TERM` 时清理临时资源。
### 3. `mktemp` 与清理
不要使用可预测的 `/tmp/app.tmp`：
```bash
temp_dir="$(mktemp -d)"
cleanup() {
    [[ -n "$temp_dir" && -d "$temp_dir" ]] && rm -rf -- "$temp_dir"
}
trap cleanup EXIT INT TERM
```
`mktemp` 避免临时文件名冲突。清理函数应简单、可重复执行，并确保变量非空。
### 4. 幂等与前置检查
```bash
install -d -m 0750 /var/lib/myapp
config="/etc/myapp/app.conf"
backup="${config}.$(date '+%Y%m%d_%H%M%S').bak"
[[ -f "$config" ]] && cp --preserve=all -- "$config" "$backup"
```
运维脚本应尽量可重复执行。破坏性操作前检查目标非空、路径范围、权限和备份结果，并记录实际修改对象。
---
## 九、调试与静态检查
语法检查，不执行脚本：
```bash
bash -n script.sh
```
跟踪执行：
```bash
bash -x script.sh
set -x
printf '仅跟踪这一段\n'
set +x
```
调试输出可能泄露密码和令牌，处理敏感变量前应关闭跟踪。
安装和运行 ShellCheck：
```bash
# Ubuntu
sudo apt install shellcheck

# Arch Linux
sudo pacman -S shellcheck

shellcheck script.sh
```
ShellCheck 能发现未引用变量、可疑条件、拆词和不安全遍历。确实需要忽略规则时应就近说明原因。
推荐日志函数：
```bash
log() {
    local level="$1"
    shift
    printf '[%s] [%s] %s\n' "$(date '+%F %T')" "$level" "$*"
}
```
日志至少包含时间、级别、对象和结果，不记录凭据。函数名用动词开头，主流程保持简短。涉及复杂数据、并发或大量 API 编排时应考虑 Python。
---
## 十、实战脚本：服务健康检查与告警占位
脚本检查 systemd 服务和 HTTP 接口，告警暂用 `logger` 写系统日志，可替换为监控平台 Webhook。
```bash
#!/usr/bin/env bash
set -Eeuo pipefail

service="nginx"
url="http://192.168.56.20:8080/health"
timeout=5
temp_body=""

usage() { printf '用法：%s [-s 服务] [-u 地址] [-t 秒数]\n' "$0"; }
log() { printf '[%s] %s\n' "$(date '+%F %T')" "$*"; }
alert() {
    local message="$1"
    log "告警：$message" >&2
    logger -t health-check -- "$message"
}
cleanup() {
    [[ -n "$temp_body" && -e "$temp_body" ]] && rm -f -- "$temp_body"
}
trap cleanup EXIT INT TERM

while getopts ":s:u:t:h" option; do
    case "$option" in
        s) service="$OPTARG" ;;
        u) url="$OPTARG" ;;
        t) timeout="$OPTARG" ;;
        h) usage; exit 0 ;;
        :) printf '选项 -%s 缺少值\n' "$OPTARG" >&2; exit 2 ;;
        \?) printf '未知选项：-%s\n' "$OPTARG" >&2; exit 2 ;;
    esac
done

[[ "$timeout" =~ ^[1-9][0-9]*$ ]] || {
    printf '错误：超时必须是正整数\n' >&2
    exit 2
}
for cmd in systemctl curl logger mktemp; do
    command -v "$cmd" >/dev/null 2>&1 || {
        printf '错误：缺少命令 %s\n' "$cmd" >&2
        exit 3
    }
done

if ! systemctl is-active --quiet "$service"; then
    alert "服务未运行：$service，主机：$(hostname)"
    exit 1
fi

temp_body="$(mktemp "${TMPDIR:-/tmp}/health-check.XXXXXX")"
if ! code="$(curl --silent --show-error --output "$temp_body" \
    --write-out '%{http_code}' --max-time "$timeout" "$url")"; then
    alert "接口无法访问：$url，主机：$(hostname)"
    exit 4
fi

if [[ "$code" != "200" ]]; then
    preview="$(tr '\n' ' ' < "$temp_body" | cut -c 1-200)"
    alert "接口异常：HTTP $code，响应：$preview"
    exit 1
fi

log "检查通过：$service，$url，HTTP $code"
```
运行：
```bash
chmod +x health-check.sh
./health-check.sh -s nginx -u "http://192.168.56.20:8080/health" -t 5
```
接入真实告警时从受限环境文件或 Secret 读取 Webhook，不要把密钥写入脚本。
---
## 十一、实战脚本：日志切割与清理
脚本压缩当前日志、清空原文件以保留 inode，并删除超过保留天数的归档。高并发服务优先使用系统 `logrotate`。
```bash
#!/usr/bin/env bash
set -Eeuo pipefail

log_file=""
archive_dir=""
keep_days=14

usage() { printf '用法：%s -f <日志> -d <归档目录> [-k 天数]\n' "$0"; }
log() { printf '[%s] %s\n' "$(date '+%F %T')" "$*"; }

while getopts ":f:d:k:h" option; do
    case "$option" in
        f) log_file="$OPTARG" ;;
        d) archive_dir="$OPTARG" ;;
        k) keep_days="$OPTARG" ;;
        h) usage; exit 0 ;;
        :) printf '选项 -%s 缺少值\n' "$OPTARG" >&2; exit 2 ;;
        \?) printf '未知选项：-%s\n' "$OPTARG" >&2; exit 2 ;;
    esac
done

[[ -n "$log_file" && -n "$archive_dir" ]] || { usage >&2; exit 2; }
[[ "$keep_days" =~ ^[0-9]+$ ]] || {
    printf '错误：保留天数必须是非负整数\n' >&2
    exit 2
}
[[ -f "$log_file" && -r "$log_file" && -w "$log_file" ]] || {
    printf '错误：日志不存在或权限不足：%s\n' "$log_file" >&2
    exit 1
}
for cmd in gzip find install; do
    command -v "$cmd" >/dev/null 2>&1 || exit 3
done

install -d -m 0750 -- "$archive_dir"
base="$(basename -- "$log_file")"
stamp="$(date '+%Y%m%d_%H%M%S')"
archive="${archive_dir}/${base}.${stamp}.gz"
temp="${archive}.tmp.$$"
cleanup() { rm -f -- "$temp"; }
trap cleanup EXIT INT TERM

log "开始归档：$log_file"
gzip -c -- "$log_file" > "$temp"
mv -- "$temp" "$archive"
: > "$log_file"
log "归档完成：$archive"

find "$archive_dir" -type f -name "${base}.*.gz" \
    -mtime "+$keep_days" -print -delete
log "过期归档清理完成"
```
实验：
```bash
mkdir -p "$HOME/shell-lab/logs" "$HOME/shell-lab/archive"
printf '模拟日志\n' > "$HOME/shell-lab/logs/app.log"
chmod +x rotate-log.sh
./rotate-log.sh -f "$HOME/shell-lab/logs/app.log" \
    -d "$HOME/shell-lab/archive" -k 7
```
多实例运行时应加 `flock`。清理范围必须限制在明确目录和模式内；可通知服务重新打开日志时，优先使用 `logrotate` 的标准机制。
---
## 十二、实战脚本：日期备份与保留策略
脚本把目录打包到备份目录，临时文件成功后再改名，校验归档并清理旧备份。
```bash
#!/usr/bin/env bash
set -Eeuo pipefail

source_dir=""
backup_dir=""
keep_days=30

usage() { printf '用法：%s -s <源目录> -d <备份目录> [-k 天数]\n' "$0"; }
log() { printf '[%s] %s\n' "$(date '+%F %T')" "$*"; }

while getopts ":s:d:k:h" option; do
    case "$option" in
        s) source_dir="$OPTARG" ;;
        d) backup_dir="$OPTARG" ;;
        k) keep_days="$OPTARG" ;;
        h) usage; exit 0 ;;
        :) printf '选项 -%s 缺少值\n' "$OPTARG" >&2; exit 2 ;;
        \?) printf '未知选项：-%s\n' "$OPTARG" >&2; exit 2 ;;
    esac
done

[[ -n "$source_dir" && -n "$backup_dir" ]] || { usage >&2; exit 2; }
[[ "$keep_days" =~ ^[0-9]+$ ]] || exit 2
[[ -d "$source_dir" && -r "$source_dir" ]] || {
    printf '错误：源目录不存在或不可读：%s\n' "$source_dir" >&2
    exit 1
}
for cmd in tar find install; do
    command -v "$cmd" >/dev/null 2>&1 || exit 3
done

source_dir="${source_dir%/}"
[[ -n "$source_dir" && "$source_dir" != "/" ]] || {
    printf '错误：本示例不备份根目录\n' >&2
    exit 2
}
install -d -m 0750 -- "$backup_dir"

parent="$(dirname -- "$source_dir")"
name="$(basename -- "$source_dir")"
safe_name="${name// /_}"
stamp="$(date '+%Y%m%d_%H%M%S')"
archive="${backup_dir}/${safe_name}_${stamp}.tar.gz"
temp="${archive}.tmp.$$"
cleanup() { rm -f -- "$temp"; }
trap cleanup EXIT INT TERM

log "开始备份：$source_dir"
tar -C "$parent" -czf "$temp" -- "$name"
mv -- "$temp" "$archive"
if ! tar -tzf "$archive" >/dev/null; then
    rm -f -- "$archive"
    printf '错误：归档校验失败\n' >&2
    exit 1
fi
log "备份完成：$archive"

find "$backup_dir" -type f -name "${safe_name}_*.tar.gz" \
    -mtime "+$keep_days" -print -delete
log "保留策略执行完成"
```
实验：
```bash
mkdir -p "$HOME/shell-lab/app/config"
printf '端口=8080\n' > "$HOME/shell-lab/app/config/app.conf"
chmod +x backup-directory.sh
./backup-directory.sh -s "$HOME/shell-lab/app" \
    -d "$HOME/shell-lab/backups" -k 14
```
本机归档不等于容灾。生产备份还应同步到独立存储、加密敏感数据并定期恢复演练；数据库应使用能保证一致性的专用备份工具。
---
## 十三、常见坑
### 1. word splitting
```bash
path="/srv/app data"
mkdir -p $path          # 错误
mkdir -p -- "$path"    # 正确
```
### 2. glob 意外展开
```bash
pattern="*.log"
printf '%s\n' $pattern     # 可能展开为文件列表
printf '%s\n' "$pattern"   # 按字面值输出
```
确实需要 glob 时启用 `nullglob` 并使用数组。
### 3. `cd` 失败后继续
```bash
cd "$work_dir" || {
    printf '无法进入：%s\n' "$work_dir" >&2
    exit 1
}
rm -rf -- ./*
```
危险操作更适合使用经校验的绝对路径，并确认目标不是空值或根目录。
### 4. 解析 `ls`
```bash
for file in $(ls *.log); do :; done    # 错误

shopt -s nullglob
for file in ./*.log; do
    printf '%s\n' "$file"
done
```
递归遍历使用 `find -print0`，不要把面向人类的 `ls` 输出当数据格式。
### 5. 使用 `eval` 或字符串保存命令
```bash
curl_options=(--fail --silent --show-error --max-time 5)
curl "${curl_options[@]}" "$url"
```
使用数组保留参数边界，不要让外部输入进入 `eval`，否则可能造成命令注入。
### 6. 忽略关键命令失败
```bash
if ! cp -- config.new /etc/myapp/app.conf; then
    printf '配置复制失败\n' >&2
    exit 1
fi
if ! systemctl restart myapp; then
    printf '服务重启失败\n' >&2
    exit 1
fi
```
不要在失败后仍输出“成功”。密码也不要放在命令行或 `set -x` 跟踪中，应使用受限配置文件或 Secret 管理。
---
## 十四、学习路线与实践任务
学习主线：先写能运行的短脚本，再补参数化和错误处理，最后接入定时任务、告警与版本评审。
### 1. 基础清单
- [ ] 创建带 shebang 的脚本，并用三种方式运行。
- [ ] 观察成功和失败命令的退出码。
- [ ] 正确处理包含空格的文件名。
- [ ] 使用 `${var:-default}` 提供默认配置。
- [ ] 使用 `[[ ]]` 完成字符串、数字和文件判断。
- [ ] 使用 `while IFS= read -r` 读取主机清单。
- [ ] 编写包含 `local` 变量的函数。
- [ ] 使用 `"$@"` 原样转发参数。
- [ ] 使用 `getopts` 实现帮助和参数校验。
- [ ] 使用 here-document 生成实验配置。
### 2. 健壮性清单
- [ ] 启用 `set -Eeuo pipefail` 并观察失败行为。
- [ ] 使用 `mktemp` 和 `trap` 清理临时资源。
- [ ] 使用 `command -v` 检查外部依赖。
- [ ] 为路径变量补充双引号和 `--`。
- [ ] 对删除目标增加非空和范围校验。
- [ ] 使用 `bash -n` 检查语法。
- [ ] 使用 ShellCheck 并理解每条告警。
- [ ] 使用 `bash -x` 定位一次错误。
- [ ] 验证失败时脚本返回非 `0`。
- [ ] 在隔离环境模拟权限不足和网络超时。
### 3. 运维实践清单
- [ ] 让健康检查支持多个服务和连续失败告警。
- [ ] 为日志切割增加 `flock` 单实例锁。
- [ ] 用 `logrotate` 重写切割任务并比较差异。
- [ ] 为备份增加磁盘空间与 SHA-256 校验。
- [ ] 把备份同步到实验主机 `192.168.56.30`。
- [ ] 编写恢复脚本并完成一次恢复演练。
- [ ] 使用 `cron` 或 systemd timer 定时运行。
- [ ] 确保日志不包含密码、令牌或私钥。
- [ ] 在 Git 中评审脚本变更。
- [ ] 为失败场景准备回滚步骤。
---
## 十五、总结
Shell 脚本的学习路径可以概括为：
```text
手动命令 -> 变量与判断 -> 循环与函数 -> 参数化 -> 错误处理 -> 定时与自动化
```
可靠脚本应明确解释器和退出码，默认引用变量，校验参数、路径、权限与依赖，使用 `trap` 清理资源，并通过 `bash -n`、`bash -x` 和 ShellCheck 检查。
当脚本能够安全重复运行、失败时给出清晰信息、成功后提供可验证结果时，它才真正从“命令集合”变成可维护的运维工具。
---
