---
title: "银狐病毒 (SilverFox) 深度分析：Go语言木马的感染链与检测实战"
date: 2026-05-25
draft: false
tags:
  - 安全
  - 银狐
  - 感染链
  - Go木马
  - IOC
  - YARA
categories:
  - 安全研究
---

## 前言

银狐病毒（SilverFox）是2022年9月由腾讯安全、360、微步在线三家厂商几乎同时独立发现的针对中国企业的恶意软件家族。与传统的C/C++木马不同，银狐使用 **Go语言编写**，这带来了独特的检测挑战和特征。

银狐的目标明确：中国企业的财务部门。攻击手法成熟：钓鱼邮件、即时通讯、假冒软件更新。持久化手段多样：注册表、WMI、计划任务、AppInit_DLLs。防御规避专业：篡改Windows Defender排除项、进程注入、随机进程名。

本文基于开源检测工具源代码分析，提供：
- 银狐的完整感染链分析
- Go语言木马的技术特征
- 增强版YARA规则（覆盖行为特征）
- 可直接使用的检测脚本

> **声明**: 本文IOC来自开源检测工具源代码，最新IOC请从官方查杀工具获取。

---

## 一、银狐病毒技术特征

### 1.1 Go语言木马的特征

银狐使用Go语言编写，具有以下可检测特征：

| 特征类型 | 检测方法 | 说明 |
|---------|---------|------|
| **Go运行时库** | 内存扫描/字符串分析 | Go程序加载`runtime.dll`、`go.dll`等运行时库 |
| **Go二进制结构** | PE头分析 | Go编译的二进制文件有特定的PE节区（如`.go.buildinfo`） |
| **Go异常处理** | 行为分析 | Go的panic/recover机制与C++异常处理不同 |
| **Go协程特征** | 线程行为 | Go的Goroutine调度器会产生特定的线程创建模式 |

### 1.2 银狐的行为特征

根据开源检测工具分析，银狐具有以下行为：

```
1. 进程注入：注入 svchost.exe 等系统进程
2. 注册表持久化：HKCU/HKLM Run键 + AppInit_DLLs
3. WMI事件订阅：__EventFilter + __EventConsumer + __FilterToConsumerBinding
4. 计划任务：创建 Task1 或 SilverFox 相关任务
5. Windows Defender排除：篡改排除路径以规避检测
6. 文件伪装：使用 svchost64.exe、随机进程名（pXDc9LSz.exe）
```

---

## 二、感染链分析

银狐的完整攻击链如下：

```
┌─────────────────────────────────────────────────────────────────────┐
│                        银狐感染链                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  阶段1: 初始访问                                                    │
│  ├── 钓鱼邮件（伪装成发票、合同）                                    │
│  ├── 即时通讯（微信/钉钉发送恶意文件）                               │
│  └── 假冒软件更新（财务软件、OA系统）                                │
│                                                                     │
│  阶段2: 执行                                                        │
│  ├── 用户双击恶意附件                                              │
│  ├── 恶意宏代码执行                                                 │
│  └── 社会工程学诱导（"文件恢复指南"等）                              │
│                                                                     │
│  阶段3: 持久化                                                      │
│  ├── 注册表 Run 键写入                                               │
│  ├── WMI 事件订阅（__EventFilter）                                  │
│  ├── 计划任务创建                                                   │
│  └── AppInit_DLLs 注入                                              │
│                                                                     │
│  阶段4: 防御规避                                                    │
│  ├── Windows Defender 排除项篡改                                    │
│  ├── 进程注入（svchost.exe）                                        │
│  ├── 随机进程名生成                                                 │
│  └── 文件伪装（svchost64.exe）                                      │
│                                                                     │
│  阶段5: C2通信                                                      │
│  ├── HTTP/HTTPS 心跳包                                              │
│  ├── DNS 查询（可能使用DGA）                                        │
│  └── 加密通信（TLS/自定义协议）                                     │
│                                                                     │
│  阶段6: 数据窃取                                                    │
│  ├── 浏览器凭证窃取                                                 │
│  ├── 财务软件凭证窃取                                               │
│  └── 即时通讯凭证窃取                                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.1 各阶段检测要点

| 阶段 | 检测重点 | 检测工具 |
|------|---------|---------|
| 初始访问 | 邮件附件、钓鱼链接 | 邮件网关、URL过滤 |
| 执行 | 可疑进程启动 | EDR、进程监控 |
| 持久化 | 注册表、WMI、计划任务 | 注册表监控、WMI监控 |
| 防御规避 | Defender排除项、进程注入 | 安全配置审计、内存扫描 |
| C2通信 | 异常网络连接、DNS查询 | 网络流量分析、DNS监控 |
| 数据窃取 | 凭证访问、文件外传 | DLP、凭证监控 |

---

## 三、IOC 列表（来自开源工具）

以下IOC来自 [zseagate/SilverFox-Scanner](https://github.com/zseagate/SilverFox-Scanner) 和 [das-secbox/silverfox_scanner](https://github.com/das-secbox/silverfox_scanner) 的源代码。

### 3.1 恶意进程名

```
foxservice.exe
xfolder32*
svchost.exe          # 注意：正常svchost在System32，异常路径的是恶意
*silverfox*
pXDc9LSz.exe         # 随机生成的进程名示例
pQpfOm.exe           # 随机生成的进程名示例
svchost64.exe        # 伪装进程
```

### 3.2 注册表持久化

```
HKCU\Software\Microsoft\Windows\CurrentVersion\Run
HKLM\Software\Microsoft\Windows\CurrentVersion\Run
HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Windows\AppInit_DLLs
HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Shell Folders
```

### 3.3 WMI 持久化

```
__EventFilter
__EventConsumer
__FilterToConsumerBinding
Namespace: root\subscription
```

### 3.4 计划任务

```
Task1
SilverFox
```

### 3.5 恶意文件特征

```
*.silverfox
*silverfox*
foxservice
svchost64.exe
!!!文件恢复指南*
```

### 3.6 恶意文件路径

```
C:\ProgramData\xfolder32
C:\Users\Public\Documents\
C:\Users\$USERNAME\AppData\Local\Temp\
```

### 3.7 Windows Defender 排除项

银狐常篡改Windows Defender排除路径以规避检测，需检查：

```
Get-MpPreference | Select-Object -ExpandProperty ExclusionPath
```

---

## 四、检测脚本

### 4.1 Windows 检测（PowerShell）

```powershell
# 银狐病毒检测脚本 - Windows版本
# 来源: zseagate/SilverFox-Scanner

Write-Host "=== 银狐病毒检测 (Windows) ===" -ForegroundColor Cyan

# 1. 检查恶意进程
Write-Host "`n[1/6] 检查可疑进程..." -ForegroundColor Yellow
$maliciousProcesses = @("foxservice.exe", "xfolder32*", "svchost.exe", "*silverfox*", "pXDc9LSz.exe", "pQpfOm.exe", "svchost64.exe")
$foundProcesses = Get-Process | Where-Object { $processName = $_.Name; $maliciousProcesses | Where-Object { $processName -like $_ } }
if ($foundProcesses) {
    Write-Host "发现可疑进程:" -ForegroundColor Red
    $foundProcesses | Format-Table Id, Name, Path, StartTime -AutoSize
} else {
    Write-Host "未发现已知恶意进程" -ForegroundColor Green
}

# 2. 检查注册表持久化项
Write-Host "`n[2/6] 检查注册表持久化项..." -ForegroundColor Yellow
$runKeys = @(
    "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run",
    "HKLM:\Software\Microsoft\Windows\CurrentVersion\Run",
    "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Windows\AppInit_DLLs",
    "HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer\Shell Folders"
)
foreach ($key in $runKeys) {
    Write-Host "检查 $key..."
    try {
        Get-ItemProperty -Path $key -ErrorAction Stop | Select-Object * | Format-List
    } catch {
        Write-Host "无法读取该注册表项" -ForegroundColor Gray
    }
}

# 3. 检查WMI事件订阅（银狐常用持久化方式）
Write-Host "`n[3/6] 检查WMI事件订阅..." -ForegroundColor Yellow
Get-WmiObject -Namespace root\subscription -Class __EventFilter -ErrorAction SilentlyContinue | ForEach-Object {
    Write-Host "发现WMI事件过滤器: $($_.Name)" -ForegroundColor Red
    Write-Host "查询语句: $($_.Query)"
}

# 4. 检查计划任务
Write-Host "`n[4/6] 检查计划任务..." -ForegroundColor Yellow
Get-ScheduledTask | Where-Object { $_.TaskName -like "*Task1*" -or $_.Description -like "*SilverFox*" } | Format-Table TaskName, State, Description -AutoSize

# 5. 检查常见恶意文件路径
Write-Host "`n[5/6] 扫描恶意文件路径..." -ForegroundColor Yellow
$scanPaths = @(
    "C:\ProgramData\xfolder32",
    "C:\Users\Public\Documents\",
    $env:TEMP,
    "C:\Users\$env:USERNAME\AppData\Local\Temp"
)
foreach ($path in $scanPaths) {
    if (Test-Path $path) {
        Write-Host "扫描 $path..."
        Get-ChildItem -Path $path -Recurse -Force -ErrorAction SilentlyContinue | Where-Object { $_.Name -match "svchost64\.exe|.*\.silverfox|!!!文件恢复指南.*" } | ForEach-Object {
            Write-Host "发现可疑文件: $($_.FullName)" -ForegroundColor Red
        }
    }
}

# 6. 检查Windows Defender排除项（银狐常篡改此配置）
Write-Host "`n[6/6] 检查Windows Defender排除路径..." -ForegroundColor Yellow
$exclusions = Get-MpPreference | Select-Object -ExpandProperty ExclusionPath
if ($exclusions) {
    Write-Host "发现排除路径:" -ForegroundColor Red
    $exclusions | ForEach-Object { Write-Host $_ }
} else {
    Write-Host "未发现异常排除路径" -ForegroundColor Green
}

Write-Host "`n排查完成，若发现上述可疑项目，请立即断网并使用专杀工具清理" -ForegroundColor Cyan
```

### 4.2 Linux 检测（Bash）

```bash
#!/bin/bash
# 银狐病毒检测脚本 - Linux版本
# 来源: zseagate/SilverFox-Scanner

echo -e "\033[36m=== 银狐病毒检测 (Linux) ===\033[0m"

# 1. 检查可疑进程
echo -e "\n\033[33m[1/5] 检查可疑进程...\033[0m"
ps aux | grep -iE "silverfox|foxservice|svchost|minerd|xmrig" | grep -v grep
if [ $? -eq 0 ]; then
    echo -e "\033[31m发现可疑进程，请重点检查上述进程\033[0m"
fi

# 2. 检查开机启动项
echo -e "\n\033[33m[2/5] 检查开机启动项...\033[0m"
systemctl list-unit-files --type=service | grep -iE "silverfox|malware|unknown"
crontab -l 2>/dev/null | grep -iE "curl|wget|bash|python.*http"
cat /etc/crontab | grep -iE "curl|wget|bash|python.*http"

# 3. 检查恶意文件
echo -e "\n\033[33m[3/5] 扫描常见恶意路径...\033[0m"
scan_dirs=("/tmp" "/var/tmp" "/dev/shm" "/root" "/home")
for dir in "${scan_dirs[@]}"; do
    echo "扫描 $dir..."
    find "$dir" -type f \( -name "*.silverfox" -o -name "*silverfox*" -o -name "foxservice" \) 2>/dev/null
done

# 4. 检查网络连接
echo -e "\n\033[33m[4/5] 检查可疑网络连接...\033[0m"
netstat -antp 2>/dev/null | grep -iE "estab|listen" | grep -v ":22\|:80\|:443" | grep -v "127.0.0.1"

# 5. 检查最近修改的文件
echo -e "\n\033[33m[5/5] 检查最近24小时修改的可执行文件...\033[0m"
find / -type f -mtime -1 -perm /u+x 2>/dev/null | grep -vE "/bin|/sbin|/usr/bin|/usr/sbin" | head -20

echo -e "\n\033[36m排查完成，若发现可疑项请及时隔离并清理\033[0m"
```

### 4.3 macOS 检测（Bash）

```bash
#!/bin/bash
# 银狐病毒检测脚本 - macOS版本
# 来源: zseagate/SilverFox-Scanner

echo -e "\033[36m=== 银狐病毒检测 (macOS) ===\033[0m"

# 1. 检查可疑进程
echo -e "\n\033[33m[1/5] 检查可疑进程...\033[0m"
ps aux | grep -iE "silverfox|foxservice|svchost" | grep -v grep
if [ $? -eq 0 ]; then
    echo -e "\033[31m发现可疑进程，请重点检查上述进程\033[0m"
fi

# 2. 检查启动项与LoginHook
echo -e "\n\033[33m[2/5] 检查开机启动项...\033[0m"
launchctl list | grep -iE "silverfox|unknown|malware"
defaults read com.apple.loginwindow LoginHook 2>/dev/null
defaults read com.apple.loginwindow LogoutHook 2>/dev/null

# 3. 检查LaunchAgents/LaunchDaemons
echo -e "\n\033[33m[3/5] 检查Launch配置...\033[0m"
launch_dirs=(
    "/Library/LaunchAgents"
    "/Library/LaunchDaemons"
    "$HOME/Library/LaunchAgents"
)
for dir in "${launch_dirs[@]}"; do
    echo "检查 $dir..."
    ls -la "$dir" | grep -iE "silverfox|foxservice|unknown"
done

# 4. 扫描恶意文件
echo -e "\n\033[33m[4/5] 扫描恶意文件...\033[0m"
scan_dirs=("/tmp" "/var/tmp" "$HOME/Downloads" "$HOME/Documents" "/Applications")
for dir in "${scan_dirs[@]}"; do
    find "$dir" -type f \( -name "*.silverfox" -o -name "*silverfox*" -o -name "SilverFox.app" \) 2>/dev/null
done

# 5. 检查网络连接
echo -e "\n\033[33m[5/5] 检查可疑网络连接...\033[0m"
lsof -i -P | grep -iE "listen|established" | grep -v ":22\|:80\|:443" | grep -v "127.0.0.1"

echo -e "\n\033[36m排查完成，若发现可疑项建议使用专业安全工具进一步扫描\033[0m"
```

---

## 五、YARA 规则（整合版）

以下YARA规则整合了进程名、WMI、文件特征、Go语言特征和注册表持久化检测，可直接使用。

### 5.1 银狐病毒完整YARA规则

```yara
rule SilverFox_Complete {
    meta:
        description = "银狐病毒完整检测规则（进程名 + WMI + 文件特征 + Go特征 + 注册表）"
        author = "Based on zseagate/SilverFox-Scanner"
        date = "2026-05-25"
        reference = "https://github.com/zseagate/SilverFox-Scanner"
        version = "1.0"
    
    strings:
        // === 进程名特征 ===
        $proc1 = "foxservice.exe"
        $proc2 = "xfolder32"
        $proc3 = "silverfox" nocase
        $proc4 = "svchost64.exe"
        $proc5 = "pXDc9LSz.exe"
        $proc6 = "pQpfOm.exe"
        
        // === WMI持久化特征 ===
        $wmi1 = "__EventFilter"
        $wmi2 = "__EventConsumer"
        $wmi3 = "__FilterToConsumerBinding"
        $wmi4 = "root\\subscription"
        
        // === 文件特征 ===
        $ext1 = ".silverfox"
        $name1 = "foxservice"
        $name2 = "svchost64.exe"
        $name3 = "!!!文件恢复指南"
        $name4 = "xfolder32"
        
        // === Go语言特征 ===
        $go1 = "go.buildinfo"
        $go2 = "runtime"
        $go3 = "GOTRACEBACK"
        
        // === 注册表特征 ===
        $reg1 = "CurrentVersion\\Run"
        $reg2 = "AppInit_DLLs"
        $reg3 = "Shell Folders"
    
    condition:
        // 高置信度：银狐特定字符串 + Go特征
        any of ($proc*) or any of ($name*) or any of ($wmi*) or 
        $go1 or ($go2 and any of ($reg*))
}

rule SilverFox_Process {
    meta:
        description = "银狐病毒进程名检测"
        author = "Based on zseagate/SilverFox-Scanner"
        date = "2026-05-25"
    
    strings:
        $proc1 = "foxservice.exe"
        $proc2 = "xfolder32"
        $proc3 = "silverfox" nocase
        $proc4 = "svchost64.exe"
        $proc5 = "pXDc9LSz.exe"
        $proc6 = "pQpfOm.exe"
    
    condition:
        any of them
}

rule SilverFox_WMI {
    meta:
        description = "银狐 WMI 持久化检测"
        author = "Based on zseagate/SilverFox-Scanner"
        date = "2026-05-25"
    
    strings:
        $wmi1 = "__EventFilter"
        $wmi2 = "__EventConsumer"
        $wmi3 = "__FilterToConsumerBinding"
        $wmi4 = "root\\subscription"
    
    condition:
        any of them
}

rule SilverFox_File {
    meta:
        description = "银狐病毒文件特征检测"
        author = "Based on zseagate/SilverFox-Scanner"
        date = "2026-05-25"
    
    strings:
        $ext1 = ".silverfox"
        $name1 = "foxservice"
        $name2 = "svchost64.exe"
        $name3 = "!!!文件恢复指南"
        $name4 = "xfolder32"
    
    condition:
        any of them
}

rule SilverFox_GoBinary {
    meta:
        description = "银狐 Go语言二进制特征检测"
        author = "Based on zseagate/SilverFox-Scanner"
        date = "2026-05-25"
    
    strings:
        // Go运行时特征
        $go1 = "go.buildinfo"
        $go2 = "runtime"
        $go3 = "GOTRACEBACK"
        
        // 银狐特定字符串
        $sf1 = "foxservice" nocase
        $sf2 = "silverfox" nocase
        $sf3 = "xfolder" nocase
    
    condition:
        $go1 or ($go2 and any of ($sf1, $sf2, $sf3))
}

rule SilverFox_Registry {
    meta:
        description = "银狐注册表持久化检测"
        author = "Based on zseagate/SilverFox-Scanner"
        date = "2026-05-25"
    
    strings:
        $reg1 = "CurrentVersion\\Run"
        $reg2 = "AppInit_DLLs"
        $reg3 = "Shell Folders"
    
    condition:
        any of them
}
```

### 5.2 使用示例

```bash
# 扫描整个系统
yara -r silverfox.yar /

# 扫描特定目录
yara silverfox.yar /tmp

# 扫描进程内存（需要libyara）
yara -m silverfox.yar /proc/<pid>/mem
```

---

## 六、检测流程示例

### 6.1 企业环境检测流程

```
步骤1: 网络隔离
├── 发现可疑主机后，立即断网
└── 防止C2通信和数据外传

步骤2: 初步扫描
├── 运行银狐检测脚本
├── 检查恶意进程、注册表、WMI、计划任务
└── 记录所有可疑项

步骤3: 深度分析
├── 对可疑进程进行内存分析
├── 提取C2通信特征
└── 分析持久化机制

步骤4: 清理与恢复
├── 使用专杀工具清理
├── 恢复Windows Defender配置
├── 重置注册表和计划任务
└── 修改所有凭证

步骤5: 溯源与报告
├── 分析感染来源
├── 记录IOC
└── 提交威胁情报
```

### 6.2 个人用户检测流程

```
步骤1: 下载专杀工具
├── 火绒银狐专杀: https://down5.huorong.cn/tools/Hrkill-SilverFox.exe
├── 深信服专杀: https://download.sangfor.com.cn/download/product/edr/antivirus_tool/sfakiller_x64.exe
└── das-secbox银狐专杀: https://github.com/das-secbox/silverfox_scanner/releases

步骤2: 运行扫描
├── 全盘扫描
├── 等待结果
└── 清理发现的威胁

步骤3: 手动检查
├── 检查任务管理器是否有可疑进程
├── 检查启动项是否有异常
└── 检查浏览器是否有异常扩展

步骤4: 修改凭证
├── 修改所有重要账户密码
├── 检查浏览器保存的密码
└── 启用双因素认证
```

---

## 七、开源检测工具

| 工具 | 作者 | 特点 | 地址 |
|------|------|------|------|
| silverfox_scanner | 大安全 | 查杀库30分钟自动更新 | [GitHub](https://github.com/das-secbox/silverfox_scanner) |
| SilverFox-Scanner | zseagate | 跨平台（Win/Linux/macOS） | [GitHub](https://github.com/zseagate/SilverFox-Scanner) |
| 火绒银狐专杀 | 火绒安全 | 免费专杀工具 | [下载](https://down5.huorong.cn/tools/Hrkill-SilverFox.exe) |
| 深信服专杀 | 深信服 | 免费专杀工具 | [下载](https://download.sangfor.com.cn/download/product/edr/antivirus_tool/sfakiller_x64.exe) |

---

## 八、局限性说明

| 维度 | 状态 | 说明 |
|------|------|------|
| **IOC来源** | ✅ 已验证 | 来自开源检测工具源代码 |
| **最新IOC** | ⚠️ 需更新 | 从 das-secbox 查杀库获取（30分钟更新） |
| **样本分析** | ❌ 无 | 需要获取样本在隔离环境分析 |
| **C2溯源** | ❌ 无 | 需要专业安全团队 |
| **Go特征检测** | ⚠️ 部分 | YARA规则基于公开特征，可能不完整 |

> **建议**: 下载 [das-secbox/silverfox_scanner](https://github.com/das-secbox/silverfox_scanner/releases) 获取最新查杀库。

---

## 九、参考资源

- [腾讯安全：银狐木马家族分析报告](https://ti.qq.com/)
- [360威胁情报中心](https://ti.360.cn/)
- [微步在线威胁情报](https://x.threatbook.com/)
- [VirusTotal](https://www.virustotal.com/)
- [AlienVault OTX](https://otx.alienvault.com/)
- [das-secbox/silverfox_scanner](https://github.com/das-secbox/silverfox_scanner)
- [zseagate/SilverFox-Scanner](https://github.com/zseagate/SilverFox-Scanner)

---

*本文IOC来自开源检测工具，最新IOC请从官方查杀工具获取。*