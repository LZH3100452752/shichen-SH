# 时尘 · 自动救砖守护 AutoRescue

一个面向 Root 设备的**启动失败自动救援模块**：装模块冲突、脚本卡死、system_server 崩溃导致的进不了桌面 / 无限重启，它会在下一次开机时自动把模块禁用掉，把机器救回来。

兼容 **Magisk / KernelSU（含 SukiSU 等衍生）/ APatch**，Android 8.0+。

---

## 一、安装

- 面具 App → 模块 → 从本地安装 → 选择本 zip → 重启
- 也可以在 TWRP / OrangeFox 等 Recovery 中直接刷入
- 模块 ID：`auto_rescue`
- 数据目录：`/data/adb/auto_rescue`（卸载后保留，重装自动沿用配置）

---

## 二、打开 WebUI

1. **没有 WebUI 的面具**：在模块卡片上点 **执行（Action）** 按钮 → 自动拉起本地服务并用浏览器打开
2. **有 WebUI 的面具**（KernelSU / APatch / MMRL / KsuWebUI 等）：直接在 App 里打开模块的 WebUI 入口
3. **命令行**：
   ```sh
   sh /data/adb/modules/auto_rescue/common/api.sh status
   sh /data/adb/modules/auto_rescue/tools/rescue.sh
   ```

> v1.1 修复：部分环境 busybox httpd 传给 CGI 的是相对路径，导致模块目录推算失败（提示"找不到 api.sh"）。现在改为固定路径 + 逐级回溯 + 按 module.prop 查找三重兜底，任何一种调用方式都能定位。

WebUI 默认端口 `8080`，被占用会自动顺延，也可以在设置里改。

---

## 三、核心救砖机制

### 启动计数判定
每次开机 `+1`，成功进入桌面并稳定 N 秒后自动清零。
连续达到阈值（默认 3 次）仍未稳定 → 判定 bootloop → 救援 → 重启。

### 分级救援（渐进，不一刀切）

| 等级 | 名称 | 动作 |
|---|---|---|
| L1 | 常规清场 | 禁用全部非白名单模块 |
| L2 | 深度清场 | 禁用全部模块（含白名单），只保留守护自身 |
| L3 | 终极隔离 | 重命名模块目录，彻底移除挂载 |

每救援一次等级 +1；如果救完还是起不来，下一次自动加重一级，直到救活为止。

### 触发条件
- 连续启动失败达到阈值
- 开机超时未 boot_completed
- 卡在早期阶段（上次停在 post-fs-data 没走到 service，典型是模块脚本死循环）
- system_server 60 秒内崩 3 次 / Zygote 反复重启
- 手动强制救援标记

### 熔断保护
达到最大等级后仍失败 → 停止自动重启，避免变成"无限重启循环"耗光电量。

### v1.1 新增救砖能力
- **一键安全模式**：移除全部模块挂载，只保留守护自身，能开机再慢慢恢复
- **黄金配置**：每次稳定开机自动记为黄金快照，误操作后一键回滚
- **新装模块自动体检**：模块列表一变化就扫描它的脚本
- **高危模块自动禁用**（可选）：体检超阈值直接禁用，把危险拦在开机之前

### 白名单
加入白名单的模块在 L1–L3 救援中不会被禁用（L4/L5 仍会处理）。**守护自身默认永久保护**。

---

## 四、v1.1 新增：脚本安全检测

三种入口：**按模块** / **按路径** / **粘贴代码**，也支持**全部模块一键体检**。

检测的危险代码包括：

| 级别 | 典型规则 |
|---|---|
| critical | 删根目录、递归删 /system /data /vendor、清空所有模块、dd 写分区、mkfs/擦除分区、向分区写零 |
| high | Fork 炸弹、循环重启、开机阶段重启、杀 zygote/system_server、删系统关键文件、篡改系统权限 |
| medium | 前台死循环、劫持 zygote、替换系统框架、操作块设备、关闭 SELinux |
| low | eval 动态执行、下载即执行、写设备文件 |

**加密/混淆识别与基础解密**：base64（含从长文本中抽取片段）、gzip、xz、bzip2、ROT13、倒序、hex/xxd、openssl。支持**多层拆解**（如 base64→gzip→明文），拆完再对明文做一遍危险代码分析，并展示解密后的代码预览。

---

## 五、v1.1 新增：重要目录 / 文件防护

**能力边界（说清楚）**：用户态做不到内核级拦截 `rm` 系统调用。本模块采取的是
**建基线 → 高频巡检 → 立刻记录 → 尽力恢复** 的策略：

- 守护清单可自定义（默认含 /system/bin、/system/framework、/system/etc、/system/lib64、/system/build.prop、/vendor、/data/adb、/data/data、DCIM 等）
- 记录每个路径的条目数基线，被删会立刻发现：**MISSING**（整目录消失）、**SHRINK**（条目数骤降 ≥30%）
- 扫描 `/proc`，记录正在执行危险删除命令的进程（PROC 事件）
- 所有事件带时间戳写入告警记录，支持导出查看
- 支持创建备份（模块开关 + 基线）并一键恢复
- 可选 `chattr +i` 只读加固（多数设备 /system 不支持，失败属正常）

---

## 六、WebUI 功能（10 个页面）

| 页面 | 内容 |
|---|---|
| 总览 | 健康度圆环、救援状态、双进度条、设备信息、模块概览、通道信息 |
| 救援 | L1–L5 手动执行、强制开机救援、一键安全模式、黄金配置回滚、救援历史时间线、解除隔离 |
| 模块 | 搜索/筛选、启用禁用、白名单、单模块体检、详情、批量操作 |
| 快照 | 创建/回滚/删除、设为黄金配置 |
| 检测 | 按模块/路径/粘贴/全部体检，风险分环形图、命中项按等级排序、解密链与解密后代码、一键处置 |
| 防护 | 守护清单与状态、巡检/重建基线/备份、告警事件流、防护设置、只读加固 |
| 冲突 | 扫描哪些模块在覆盖同一个系统文件 |
| 日志 | 运行日志（3 秒自动刷新）+ root 命令终端 |
| 设置 | 30 项开关与数值 |
| 工具 | 重启/软重启/Recovery/Fastboot、诊断报告、风险体检、黄金配置、关于 |

**流畅度优化**：分页面按需渲染、内容未变不写 DOM、搜索防抖 + 增量更新列表、rAF 批量写入、去掉全屏 filter/conic 重绘动画、卡片布局隔离、尊重系统"减少动画"设置。

---

## 七、常用配置

| 键 | 默认 | 说明 |
|---|---|---|
| `boot_threshold` | 3 | 连续几次启动失败触发救援 |
| `confirm_delay` | 90 | 进桌面后稳定多少秒才算成功 |
| `boot_timeout` | 180 | 开机超时秒数 |
| `max_round` | 5 | 最大救援等级 |
| `guard_enable` | 1 | 系统目录防护巡检 |
| `guard_interval` | 30 | 防护巡检间隔（秒） |
| `guard_proc_scan` | 1 | 危险删除进程检测 |
| `auto_scan_new` | 1 | 新装模块自动体检 |
| `auto_quarantine_danger` | 0 | 高危模块自动禁用 |
| `danger_score` | 60 | 高危判定阈值 |
| `webui_port` | 8080 | WebUI 端口 |
| `allow_terminal` | 1 | WebUI 命令终端 |

---

## 八、已经起不来怎么办

1. 长按电源键强制重启
2. 若仍卡住：`adb shell touch /data/adb/auto_rescue/force_rescue` 再重启
3. 进 Recovery 后：
   ```sh
   adb shell sh /data/adb/modules/auto_rescue/tools/rescue.sh
   ```
   菜单里有禁用 / 隔离 / 快照回滚 / 脚本扫描 / 防护巡检
4. 极端情况：
   ```sh
   adb shell mv /data/adb/modules /data/adb/modules.bak
   ```

---

## 九、目录结构

```
auto_rescue/
├── module.prop
├── post-fs-data.sh       早期启动检测 + 救援判定 + 熔断
├── service.sh            启动看门狗 / WebUI / 防护基线 / 新模块体检
├── boot-completed.sh     开机完成标记 + 通知
├── action.sh             「执行」按钮 → 拉起 WebUI
├── uninstall.sh          卸载时自动恢复被禁用的模块
├── common/
│   ├── functions.sh      公共函数库
│   ├── rescue.sh         分级救砖引擎 + 熔断
│   ├── watchdog.sh       常驻守护(超时/崩溃/防护巡检/新模块体检)
│   ├── scan.sh           脚本安全检测 + 基础解密引擎
│   ├── guard.sh          重要目录/文件防护引擎
│   └── api.sh            WebUI 后端
├── tools/rescue.sh       Recovery / adb 手动救援菜单
└── webroot/              WebUI（index.html + css + js + cgi-bin）
```
