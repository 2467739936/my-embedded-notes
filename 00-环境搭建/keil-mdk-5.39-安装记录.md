# Keil MDK 5.39 + STM32F103 环境搭建记录

- 机器：Windows 11（10.0.26200），普通用户 `A`（Administrators 组，UAC 开启）
- 目标：能编译、烧录、调试 STM32F103C8T6（江协科技课程同款）
- 完成日期：2026-11-20

## 最终装了什么

| 组件 | 版本 / 位置 | 说明 |
|---|---|---|
| Keil MDK-ARM | 5.39 → `C:\Keil_v5` | 安装包 `MDK539.EXE` 957,603,928 字节 |
| 授权 | MDK Community（非商业免费） | 产品码 `KEMDK-COM0`，2033-05-31 到期 |
| ARM Compiler | ArmClang V6.21 | **5.39 已不再自带 AC5**，老工程改选 V6 |
| STM32F1xx DFP | 2.4.1 | 50,237,740 字节 |
| ARM CMSIS | 5.9.0 | 34,129,867 字节，**提供 `core_cm3.h`** |
| ST-Link 驱动 | ST 官方 WinUSB（oem31.inf） | 见下方"坑 3" |
| CH340 驱动 | Windows 自带 usbser | 插上就认成 `COM3` |

## 下载源（本机实测，国内网络）

| 用途 | 地址 |
|---|---|
| MDK 安装包 | `https://armkeil.blob.core.windows.net/eval/MDK539.EXE` |
| DFP 包 | `https://keilpack.azureedge.net/pack/Keil.STM32F1xx_DFP.2.4.1.pack`（直连，**不要**走 www.keil.com 的 302） |
| CMSIS 包 | `https://keilpack.azureedge.net/pack/ARM.CMSIS.5.9.0.pack` |
| 江协资料 | 百度网盘 `https://pan.baidu.com/s/1h_UjuQKDX9IpP-U1Effbsw` 提取码 `dspb`，解压密码 `32` |

- 微软 Azure 源单连接只有 **21–26 KB/s**，957MB 要下 10 小时
- 改用 **32 线程分段并行**：实测 **1.03 MB/s**，14 分钟下完
- 脚本思路：`curl -r start-end` 分段下载 → 校验 → `cat` 合并（脚本 `get_seg.sh`）

## 安装过程（踩坑记录）

### 坑 1：MDK539.EXE 无法静默安装

- 外壳是 Inno Setup，但内部真实安装器是**自绘 Delphi 向导**，`/VERYSILENT`、`/S` 全部无效，会停在 Customer Information 页
- 解决：用 UIA 自动化读控件树填表（`First Name` id=1016、`E-mail` id=1019、`Next` id=1），或者**直接手动点**（更省事）
- 填完的信息写进 `C:\Keil_v5\TOOLS.INI` 的 `[UV2]` 段

### 坑 2：`armlm activate -product MDK` 报 `No license found with seats available`

- 服务器上的正确产品码是 **`KEMDK-COM0`**，不是 `MDK`
- 正确命令：
  ```bash
  "C:\Keil_v5\UV4\armlm\armlm.exe" activate -product KEMDK-COM0 -server https://mdk-preview.keil.arm.com
  ```
- 或用 `armlm inspect -server https://mdk-preview.keil.arm.com` 先看服务器上有哪些产品码
- 授权是**用户级**：需要先在 https://www.keil.arm.com/ 注册账号（邮箱验证码），然后 `File → License Management → User-Based License` 或在浏览器登录
- 本地缓存 **7 天**，过期需联网续期（不是一次性永久离线）

### 坑 3：ST-Link 驱动 `dpinst_amd64.exe /S` 失败

- 现象：`DPINST.LOG` 里 `DriverPackagePreinstallW` 返回 **`0xE0000242`**，设备状态仍是 Error
- 解决：改用 pnputil（管理员 PowerShell）
  ```powershell
  pnputil /add-driver "C:\Keil_v5\ARM\STLink\USBDriver\stlink_dbg_winusb.inf" /install
  ```
  成功后设备变为 `STM32 STLink`，`Class=USBDevice`，`Service=WinUSB`
- 设备 ID：`USB\VID_0483&PID_3748`（ST-Link V2）

### 坑 4：编译报 `core_cm3.h: No such file or directory`

- 原因：DFP 2.4.1 的 `Device/Include` 里只有 `stm32f10x.h`，**CMSIS 内核头文件在单独的 CMSIS Pack 里**
- 解决：装 `ARM.CMSIS.5.9.0.pack`，并在工程 IncludePath 加入
  `C:\Keil_v5\ARM\PACK\ARM\CMSIS\5.9.0\CMSIS\Core\Include`

### 坑 5：`Target 'xxx' uses RTOS support that is part of a Legacy Pack`

- 原因：工程文件里残留 `<mOS>1</mOS>`（模板自带的 RTOS 标记）
- 解决：改成 `<mOS>0</mOS>`

## 验证

- `blink.hex` 烧进板子，PC13 严格 1Hz 闪烁
- 构建日志：`Code=4498 RO-data=... 0 Error(s), 0 Warning(s)`
- 命令行验证授权是否生效：构建日志里应有
  `License Information: ... User-based license: Keil MDK Community`

## 有用的命令速查

```bash
# 编译（退出码可信）
UV4.exe -b project.uvprojx -j0 -o build.log

# 烧录（退出码不可信！必须看日志）
UV4.exe -f project.uvprojx -j0 -o flash.log

# 查 ST-Link 设备
powershell "Get-PnpDevice -PresentOnly | ? {$_.InstanceId -match 'VID_0483'}"

# 查串口
powershell "[System.IO.Ports.SerialPort]::GetPortNames()"
```
