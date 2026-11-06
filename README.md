# SystemHelper Debloater

> 用一个 0 字节的空文件，让运营商预装的终端管理客户端从系统里消失。
>
> **Systemless · 完全可逆 · 不写文件系统分区**

---

## 这是什么

一个小到不能再小的 Magisk 模块。整个模块只有两个文件：

```
module.prop                                           模块元数据
system/priv-app/SystemHelper/SystemHelper.apk         0 字节空文件（覆盖用）
```

它做的事只有一件：让 `com.mobiletools.systemhelper`（应用名 SystemHelper）这个预装在小米中国大陆官方 ROM 里的**运营商终端管理（DM, Device Management）客户端**，在系统里不再存在。

不修改系统分区，不写入任何二进制，不动 SELinux，不装任何服务。删掉模块重启，一切原样回来。

---

## 它移除的是什么

`com.mobiletools.systemhelper` 是小米国行 ROM 预装的运营商 DM 客户端，服务于中国联通的终端管理协议。

先澄清三件容易被误解的事：

**1. 它不是病毒，也不是渠道商偷偷塞的。**

它的签名证书与 `com.xiaomi.account`（小米账号）、`com.android.settings`（系统设置）**完全一致**——同一张 OEM 平台签名密钥。第三方拿不到这张密钥，只有厂商自己能签发。所以它是随官方 ROM 一起出厂的合法组件，不是移植包作者或渠道商的私货。

**2. 几乎所有国行手机都有同类组件。**

这是入网合规与运营商渠道协议的产物。每家 OEM 都有自己的实现，包名和路径各不相同——你可以在任何一台国行机上找到结构相似的家伙。它属于"买的手机里有一间运营商的房间"这类行业惯例，不是某一家的问题。

**3. 它按 SIM 卡状态决定是否活跃。**

插联通卡时激活，不插联通卡时长期休眠。一台从未插过联通卡的机器上，它可能一个字节都没上报过；插着联通卡的机器上，它在开机、换卡、连网时都会醒来。

**它为什么不影响系统：** 该组件不在系统关键链路上。实测移除后开机、通话、数据网络、OTA 结构全部正常——它联网注册失败时系统毫无反应，这本身就是"非关键"的证据。

---

## 为什么想让它消失

以下内容来自对 APK 的静态分析（`classes.dex` + `AndroidManifest.xml` + 原生库），以及设备数据目录的检查。

### 触发时机与行为

| 触发时机 | 行为 | 代码依据 |
|---|---|---|
| **开机** | 向 `http://dm.wo.com.cn:18080/register` 发起注册上报（**明文 HTTP**） | `OnBootReceiver` |
| **插拔 / 更换 SIM 卡** | 立即重新上报——运营商据此判断机器是否仍在使用本网卡 | `SIMStateReceiver` |
| **网络变化** | 连上联通网络或 WiFi 时触发上报 | `ConnectStateReceiver` |
| **定时** | JobScheduler + 精确闹钟周期性执行注册 | `ScheduleRegisterReceiver` / `RegisterJobService` |
| **信号采集** | 持续收集信号与基站信息（持有特权级电话状态权限） | `GetSignalInfoReceiver` |
| **应用清单** | 上报已安装应用列表（`QUERY_ALL_PACKAGES` + 使用统计权限） | `AppInfoRegisterReceiver` |
| **位置** | 内嵌第三方定位 SDK（`liblocSDK8b.so`，百度定位），持有后台精确定位权限 | 原生库 |

### 实测在设备上发现的数据

在 `shared_prefs` 中确认存在：

- **SIM 卡 ICCID 明文记录**（卡 1 为实卡 ICCID，卡 2 为空占位）
- **精确到街道门牌栋号的定位缓存**（base64 编码，含时间戳）——定位 SDK 反查出的完整地址串

### 它**不会**做的事

同样重要：静态分析中**没有发现任何远程指令通道**。

- 全包只有一个 `/register` 端点，搜索不到 wipe / lock / 静默安装 / 指令下发类代码；
- 没有短信、通讯录、通话记录权限。

所以它**不是可被远程操控的后门**，本质是一个"单向汇报器"：只写不读，只上报不执行。但上述采集在隐私层面依然是实质性的——你的 SIM 身份、物理位置和全部已安装应用，每换一次卡、每连一次网都在往外送一次。

**是否接受这种采集，是每个用户自己的判断题。这个模块只是把选择权交回你手里。**

---

## 兼容性

| 项目 | 说明 |
|---|---|
| **已验证机型** | 小米 / Redmi 国行 ROM（HyperOS / MIUI），路径 `/system/priv-app/SystemHelper/SystemHelper.apk` |
| **未验证机型** | 其他 OEM。路径不匹配时模块为空操作——**安全，但无效**（不会报错、不会崩溃、不会变砖） |
| **运行环境** | Magisk 20.4+ / KernelSU / APatch，任一支持 magic mount 的实现 |
| **Android 版本** | 无限制（纯文件覆盖，不依赖版本 API） |
| **完整性检测** | 不影响 Play Integrity / SafetyNet——不修改任何参与校验的分区 |

---

## 安装

### 推荐顺序（含数据清理）

模块只让应用"消失"，**不会清除它已经采集的历史数据**。所以正确的顺序是：先清数据 → 再装模块 → 重启。

```bash
# 1. 先清理历史数据与冻结（此时应用还在，才能 clear）
adb shell su -c "pm clear com.mobiletools.systemhelper"
adb shell su -c "pm disable-user --user 0 com.mobiletools.systemhelper"

# 2. 推送并安装模块
adb push systemhelper_debloater-v1.0.zip /data/local/tmp/
adb shell su -c "magisk --install-module /data/local/tmp/systemhelper_debloater-v1.0.zip"

# 3. 重启生效
adb reboot
```

### 图形界面安装

`Magisk app → 模块 → 从本地安装 → 选择 zip → 重启`

KernelSU / APatch 用各自的模块安装入口，原理完全相同。

---

## 验证

```bash
# 应为空输出（无匹配）
adb shell pm list packages | grep mobiletools

# 应返回空（包已不可见）
adb shell pm path com.mobiletools.systemhelper

# 数据目录应为空或不存在
adb shell su -c "ls /data/data/com.mobiletools.systemhelper/"
```

---

## 卸载 / 恢复

`Magisk app → 模块 → 移除` → 重启。

原始 APK 从未被修改，系统分区一个字节都没动。应用会原样回来，包括它原有的全部权限与监听器。

**恢复后建议：** 如果你不打算再让它工作，可以只用 `pm disable-user` 冻结它（随时 `pm enable` 解冻），比模块更轻量——代价是它仍占着 `/system` 的空间。

---

## 原理

Magisk 的 magic mount 只能**覆盖**，没有"删除"这个动作。所以在不动系统分区的前提下删除一个系统应用，标准手法是：在模块目录里放一个同路径的**空文件**，开机时 bind mount 盖到真实路径上。

```
模块内    :  system/priv-app/SystemHelper/SystemHelper.apk      (0 字节)
                  ↓  magic mount 覆盖
真实路径  :  /system/priv-app/SystemHelper/SystemHelper.apk     (被遮住)
                  ↓  PackageManager 扫描系统分区
读到 0 字节 → 解析失败 → 静默跳过 → 应用不存在
```

系统不会报错，只会忽略一个读取失败的包。日志里可能有一条 warning，无任何实际影响。

**这就是为什么模块里那个 `.apk` 是 0 字节——它不是应用，是封条。** 门（真实文件）还在，但谁也进不去了。

---

## 风险与免责

| 情况 | 说明 |
|---|---|
| **系统稳定性** | 移除后开机、通话、数据网络、OTA 结构均正常（实测）。风险低 |
| **官方 OTA** | OTA 可能把它装回来；本模块在 OTA 后依然生效（按路径覆盖），通常无需重装 |
| **合约机 / 补贴机** | 若你的机器属于合约机、话费补贴机，运营商侧可能有终端在线校验的履约要求。移除可能影响合约判定，**请先了解条款** |
| **保修** | systemless 修改不改系统分区，多数情况下不影响保修判定，但不做任何保证 |
| **数据丢失** | 本模块不写用户数据。唯一的破坏性操作是你手动执行的 `pm clear`（不可逆，但只清这一个应用的数据） |
| **责任** | 刷机有风险，使用者自行承担。作者不对任何数据丢失、设备损坏、合约纠纷负责 |

---

## 移植到其他机型

只需改两处：**目录路径** 和 **空文件名**。

### 1. 找到目标应用的实际路径

```bash
# 方法 A：已知包名
adb shell pm path <包名>

# 方法 B：扫系统分区找可疑目录
adb shell ls /system/priv-app/ /system/app/ /system_ext/priv-app/
```

### 2. 在模块目录里重建同样的相对路径，放同名空文件

```bash
mkdir -p system/priv-app/<文件夹名>
touch system/priv-app/<文件夹名>/<APK文件名>
```

注意路径必须与真实路径**逐级对应**（`/system/priv-app/X/Y.apk` → `system/priv-app/X/Y.apk`）。

### 3. 改 `module.prop` 后重新打包

```bash
zip -r -X ../my_debloater.zip module.prop system/ -x "*.DS_Store"
```

### 怎么找出自己机器上的运营商 DM 客户端

**不要盲猜包名。** 按这个顺序查：

```bash
# 1. 看系统分区里有哪些非标准预装应用
adb shell ls /system/priv-app/ /system/app/ /system_ext/priv-app/

# 2. 看谁持有"电话状态 + 定位 + 应用清单"这种组合权限
adb shell su -c "dumpsys package <包名> | grep -A40 'requested permissions:'"

# 3. 确认它是不是平台签名（与系统设置同证书 = OEM 官方组件）
adb shell su -c "dumpsys package <包名> | grep -A1 signatures"
adb shell su -c "dumpsys package com.android.settings | grep -A1 signatures"
```

**第 3 步最关键。** 同证书说明是 OEM 官方预装；**不同证书**意味着这可能是第三方注入的东西——那种情况下处理方式要更谨慎（先做完整审计，而不是直接套用本模块）。

### 不要删的东西

| 类型 | 例子 | 为什么 |
|---|---|---|
| 电话框架 | `com.android.phone` | 承载 RIL、SIM 状态机、通话路由，删了等于拆电话 |
| 系统 UI | `com.android.systemui` | 状态栏、锁屏、导航栏、SIM PIN 输入框 |
| 设置 | `com.android.settings` | 移除后无法调整任何系统设置 |
| 包安装器 | `com.android.packageinstaller` | 移除后无法安装任何应用 |
| 任何 `com.android.*` 系统核心 | — | 通用原则：**这进程崩了，你丢的是手机，不是数据** |

---

## 已知限制

- **单一路径**：只覆盖一个目录，不做多机型自动探测。多机型适配需手动改（见上）
- **不清数据**：历史采集数据需要手动 `pm clear`，模块不做这件事
- **不处理残留**：本应用没有常驻服务/账号结构，因此无需额外处理；若移植到有常驻服务的同类组件，需自行检查 `service` 与 `account` 残留
- **不隐藏模块自身**：Detected by nothing——它不注入任何进程，理论上无检测面

---

## 版本历史

### v1.0 — 2026-11-06

- 首个版本
- 覆盖路径：`/system/priv-app/SystemHelper/SystemHelper.apk`
- 附 `pm clear` / `pm disable-user` 清理流程说明

---

## License

MIT

---

## 免责声明

本模块仅用于个人设备的隐私管理。移除运营商预装组件可能影响你与运营商之间的合约履约判定，请在了解相关条款后自行决定是否使用。

作者与任何运营商、设备厂商均无关联。本项目不提供、不分发任何厂商固件或私有密钥。
