# SystemHelper Debloater

> 用一个 0 字节的空文件，让运营商预装的监控模块从系统里消失。
>
> **Systemless · 完全可逆 · 不写文件系统分区**

---

## 这是什么

一个小到不能再小的 Magisk 模块。整个模块只有两个文件：

```
module.prop                                           模块元数据
system/priv-app/SystemHelper/SystemHelper.apk         0 字节空文件（覆盖用）
```

它做的事只有一件：让 `com.mobiletools.systemhelper`（应用名 SystemHelper）这个预装在中国大陆各个品牌 ROM 里的**监控模块**，在系统里不再存在。

不修改系统分区，不写入任何二进制，不动 SELinux，不装任何服务。删掉模块重启，一切原样回来。

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

- **SIM 卡 ICCID 明文记录**
- **精确到街道门牌栋号的定位缓存**（base64 编码，含时间戳）——定位 SDK 反查出的完整地址串

**是否接受这种采集，应该是每个用户自己的判断题。这个模块只是把选择权重新交回你手里。**

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

---

## 风险与免责

| 情况 | 说明 |
|---|---|
| **系统稳定性** | 移除后开机、通话、数据网络、OTA 结构均正常（实测）。风险低 |
| **官方 OTA** | OTA 可能把它装回来；本模块在 OTA 后依然生效（按路径覆盖），通常无需重装 |
| **数据丢失** | 本模块不写用户数据。唯一的破坏性操作是你手动执行的 `pm clear`（不可逆，但只清这一个应用的数据） |
| **责任** | 刷机有风险，使用者自行承担。作者不对任何数据丢失、设备损坏、合约纠纷负责 |

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
