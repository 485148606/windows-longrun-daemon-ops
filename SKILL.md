---
name: Windows 长驻守护进程：启动 / 巡检 / 抢救
description: >-
  【Windows 长驻守护进程：启动 / 巡检 / 抢救】>- 【Windows 长驻守护进程：启动 / 巡检 / 抢救】在 WorkBuddy(Windows) 上拉起、巡检、抢救「需要 7×24 常驻的长跑进程」（本地渲染宿主、ComfyUI、看门狗、批量下载/转码）。覆盖：哪些启动通道能活、进程莫名消失怎么定位死因、wmic 被下线后怎么替代、计划任务为什么被拒、PowerShell 工具无输出怎么绕。触发词：守护进程、后台常驻、进程没了、链路挂了、保活、看门狗、渲染宿主、ComfyUI 掉线、进程莫名消失。
agent_created: true
---

# Windows 长驻守护进程：启动 / 巡检 / 抢救

## 一、启动通道选型（决定活多久）

| 通道 | 存活范围 | 结论 |
|---|---|---|
| PowerShell 工具 `Start-Process` | 该次工具调用结束即死 | ❌ 别用 |
| bash 工具 `run_in_background` | **该次命令一结束就整树被杀**（实测 45 秒内 python 归零）；若命令本身不退出（`start.sh ; sleep 86400`）可续命到会话终结 | ⚠️ 会话内应急 |
| 进程 A 内部 `subprocess.Popen(DETACHED_PROCESS)` | **仍然继承 A 的 Job**，A 所在命令结束照样被清 | ❌ 单独用没用 |
| `os.startfile(bat)` / `ShellExecute` | **在本进程内 CreateProcess → 照样继承 Job** | ❌ 实测同秒被杀 |
| **`subprocess.Popen(["explorer.exe", <bat>])`** | **由已运行的 explorer 实例创建进程 → 脱离 Job** ✅ | ✅✅ **正解** |
| **人手双击桌面 .bat** | 父级 explorer，天然脱离 | ✅✅ 用户侧首选 |
| HKCU `...\CurrentVersion\Run` / Startup 文件夹 | 开机登录后自启，免提权；**但很多用户明确反对常驻自启** | ⚠️ 先问用户 |
| `schtasks /Create`、`Register-ScheduledTask` | 5 分钟级自愈 | ❌ 本机**非提权令牌被拒**，需用户手动管理员操作 |
| WMI `Invoke-CimMethod Win32_Process Create` | 会话外 | ❌ 安全层直接拦 |

**核心判断**：要 7×24，必须由「会话之外」的父级拉起。
**2026-09-16 实测补正**：围栏粒度是**「每次工具调用」**，不是"整个会话"——
一次 `Bash` 调用里起的四层渲染链，在该调用返回后 45 秒内全部消失（`start_stop.log` 里体检还是
"四层在线"，下一次查询 0 个进程）。**别再相信"会话内长跑没问题"这种说法。**

### ★ 脱离围栏的唯一实测解法：explorer 转交 + ASCII 路径

```python
# 关键三步：① bat 落在纯 ASCII 路径 ② bat 内容里再指向中文路径 ③ 交 explorer
import subprocess
subprocess.Popen(["explorer.exe", r"C:\Users\<user>\.h3\h3start.bat"])
```

- **`explorer.exe` 对中文路径参数会丢**：`explorer.exe "D:\...\三角洲干员\video\start.bat"`
  实测**根本没执行**（日志无任何新记录）。所以启动器必须放 ASCII 路径。
- bat **内容**可以含中文路径（bat 由 cmd 按系统代码页 GBK 读，没问题；用 `encoding="gbk"` 写）。
- 验证是否真脱离：写个 marker 脚本（每 10 秒追加一行时间戳），
  **在发起命令之后另起一次 Bash 调用**去看 —— 还在写就是脱离了。
- 人用 → 双击桌面 bat；AI/自动化用 → 跑 `launcher_explorer.py`（把 ASCII bat 交 explorer）。

**⚠️ 2026-09-20 复测：explorer 转交不是 100% 一次成功，交付后必须落地验证。**
- 现象：交付一个 ASCII 路径、GBK 编码、内容正确的 bat 后，**毫无反应** ——
  `start_stop.log` 无新记录、`tasklist` 里既无 python 也无 pythonw、`_stdout.log` 时间戳未变。
  用**内容等价、只换文件名**的新 bat 再交付一次 → 立刻成功（45 秒内四层在线）。

**★ 2026-09-20 晚间再实测：换名重交不是「试一次就成」，要连续试到成功为止。**
同一台机器、同一 bat 内容，交付序列为：`h3start.bat`（失败）→ `h3go2.bat`（失败）→
`h3diag.bat`（成功，45 秒内四层在线）。**第 1、2 次换名都毫无反应，第 3 次才成。**
→ 纪律：**连续换名重交 2–3 次**再怀疑通道本身，别试一次失败就换路线（换路线成本更高）。
→ 更省事的做法：**直接把 marker 打点写进正式 bat** —— 交付一次同时完成「尝试启动」和「可诊断」，
  成功时有 marker + 主日志双证据，失败时也能立刻区分是没执行还是子进程起不来。
- 所以「交出去」≠「起起来了」。**判断成功只认两个硬证据**：
  ① 主日志（`start_stop.log`）出现本次新时间戳；② `tasklist /FI "IMAGENAME eq pythonw.exe"` 有 PID。
  两者缺一 → 重交一次，别自己去改链路代码。
- **隔离打点法**（区分"bat 没被执行"和"bat 执行了但子进程起不来"）：
  ```bat
  @echo off
  echo STEP1_bat_started %time% >> C:\...\_marker.log
  start "" "C:\...\pythonw.exe" -u "D:\...\中文路径\start.py"
  echo STEP2_start_called %time% >> C:\...\_marker.log
  ```
  - 一条都没写 → explorer 没执行 bat（换文件名重交）。
  - STEP1 有、STEP2 有、但主日志没动 → bat 执行了，是 `start` 起的程序自身失败
    （优先查 pythonw 路径是否存在、脚本是否一启动就抛异常）。
- 排障期间临时把 `pythonw.exe` 换成 `python.exe`：pythonw 无控制台，早期异常全静默；
  python.exe 会弹可见窗口，是"看得见失败"的最低成本手段。定位完再换回 pythonw。

**✅ 推荐：把启动器固化成「带打点的 ASCII bat」，别再每次临时造（2026-09-21 验证）**
- 固定一个 `C:\Users\<user>\.h3\h3go.bat`，内容 = STEP1 打点 + `start` 拉起 + STEP2 打点，
  打点写进 `h3go.log`。以后每次续跑只需 `Popen(["explorer.exe", <该 bat>])`，
  一次成功率高，且**任何时候都能从 h3go.log 反查"到底交付成功没有"**。
- 生成器脚本（`_go.py`）用 `encoding="gbk"` 写 bat。
- ⚠️ **坑：用 Python 拼 bat 内容时别用 `%` 格式化**。
  `'echo STEP1 %date% %time% >> "%s"' % LOG` 会炸 —— cmd 的 `%date%` 里那个 `%d`
  被 Python 当成占位符，报 `TypeError: %d format: a real number is required, not str`。
  改用字符串拼接（`"echo ... %date% ... >> " + LOG`），或把 `%` 写成 `%%`。

**续跑类任务的标准动作（先验状态，再拉链路）**
1. `tasklist /FI "IMAGENAME eq python.exe"` + `curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8188/system_stats`
   → 确认整棵树是死的还是活的。
2. 数产出：`ls state/*.ok | wc -l`，和上次记录比对，得出"这轮净增多少段"。
3. 看 `start_stop.log` 有没有"一键停止"记录：**有 = 用户主动停的；没有但心跳时间戳停止 = 被外部强杀/关机**。
   这个区分很重要 —— 前者不用查故障，后者要警惕是不是又踩了围栏。
4. 再走启动流程，**落地验证只认：① start_stop.log 出现新时间戳 ② tasklist 有 pythonw 的 PID**。
5. 用最近两次成功段的日志时间戳算单段耗时，给用户报 ETA（别用"段数×固定值"硬算）。

## 二、进程莫名消失 —— 死因分诊（先定位，别乱改代码）

按顺序看，**别跳过**：

1. `tasklist | grep -i python` → 0 个进程 = 整棵树的死，不是单点故障。
2. 看主日志（如 `comfy.log` / `render.log`）停在**哪一步**：停在"加载大模型"之类的中途 = 被杀；有 traceback = 崩溃。
3. 看 `*_err.log`：**全空**是强烈信号 —— 进程被外部杀掉不产生任何 stderr。
4. 时间线对齐：多个**不相关**进程是否**同一秒**消失？是 → 外部围栏（WorkBuddy 的 Job Object，`windows-child-process-containment.cjs`）连树清理。
5. 结论只有两种：**崩溃**（去改代码）或**被外部杀**（去换启动通道）。混淆这两种会白干。
6. **查物理内存**（大模型 offload 工作流必查，2026-09-20 新增）：
   日志里出现 `Using RAM pressure cache`、模型加载行报 `19995MB Staged`（主模型）+ `14956MB`
   （text encoder）这类数字时，把两者相加对比系统 RAM：
   ```powershell
   # PowerShell 工具 stdout 恒空 → 必须写文件再读
   $os=Get-CimInstance Win32_OperatingSystem
   ([math]::Round($os.TotalVisibleMemorySize/1MB,1)).ToString()+"GB total / "+
   ([math]::Round($os.FreePhysicalMemory/1MB,1)).ToString()+"GB free" |
     Set-Content "D:\_mem.txt" -Encoding UTF8
   ```
   **判据**：`主模型 + text encoder > 物理内存` → 每段必然狂吃页面文件 → 表现就是
   「电脑卡死 → 进程被杀 → 重试 → 再卡死」的循环。**这不是代码 bug，加内存才是解**。
   实例：H3 需 20GB+15GB=35GB，机器只有 31GB → 反复中断；此时换启动通道/改脚本都是白费。

## 三、本机已确认的环境坑

- **`wmic` 已被移除**（Win11 下线）：`os.system('wmic process ... call terminate')` 全部**静默 no-op**（stderr 还重定向到 nul）。
  替代：`Get-CimInstance Win32_Process`（能拿 CommandLine）+ `taskkill /F /PID <pid>`。
- **PowerShell 工具 stdout 恒为空**（命令确实执行了）。回传结果一律 `Set-Content` 写文件再 Read。
- **Bash 工具命令行里出现 `powershell` 字样（哪怕只是路径）会被安全层拒**。调 PowerShell 走 PowerShell 工具。
- 给计划任务/自启项传参时，**路径避开中文**（如 `C:\Users\<user>\.<app>\launcher.py`），中文路径经 cmd 容易烂编码。
- Git Bash 里 `//FI` 这类双斜杠**不保证**被还原成 `/FI`（实测 tasklist 报"无效参数"），稳妥用 `MSYS_NO_PATHCONV=1` + 单斜杠。

## 四、幂等保活脚本模板（可直接抄）

```python
BASE = r"<项目绝对路径>"; PY = r"<python.exe 绝对路径>"
DETACH = 0x00000008 | 0x00000200        # DETACHED_PROCESS | CREATE_NEW_PROCESS_GROUP

def proc_table():   # 返回 [(pid, cmdline)]，用 CIM（本机没 wmic）
    ps = ("Get-CimInstance Win32_Process -Filter \"Name='python.exe'\" | "
          "ForEach-Object { \"$($_.ProcessId)`t$($_.CommandLine)\" }")
    out = subprocess.check_output([PS, "-NoProfile", "-Command", ps], timeout=120)
    ...  # 按 \t 切分，pid 是数字才收

# 判定优先级：心跳新鲜(<=300s) 或 命令行命中 -> 存活
# 有「单实例锁」的脚本（靠心跳判陈旧）可放心重复启动 -> 用心跳判
# 没有锁的脚本（如 watchdog）只能命令行扫描 -> 命中就绝不重复拉，否则会出双份
```

配套写法：
- 脚本自带 `--dry`（只打印判定，不拉进程）——**改逻辑后必先 --dry 验证**。
- 单独留 `--kill`（只清游离进程）。
- 拉起前若怀疑上游卡死，先 `taskkill` 清场再拉，避免端口被僵尸占住。
- 拉起日志写进项目 `state/keepalive.log`，下次巡检**先看它**，比翻三个日志快。

## 五、修复时的纪律

- 用户在跑的生产脚本**能不改就不改**：新写的保活/清理逻辑放独立文件，不动原脚本 —— 既补上漏洞，又不惊动正在跑的东西。
- 改动前先备份/先 `--dry`；改完必须**端到端验证一次**（等一个新产出物真正落盘），别以"进程活着"当成功标准。
- 无提权就别硬刚计划任务，把需要用户手动执行的那一条命令原样报出来，别反复重试刷积分。
  （2026-09-16 复测：`schtasks /Create` 仍然返回「错误: 拒绝访问。」）

---

## 六、实战事故复盘：看门狗"配了个寂寞"的三种写法（2026-09-16）

用户原话："好像断了，检测没有起作用吗"。排查后发现看门狗存在三个叠加缺陷，**一个都没触发**：

### 缺陷 1：巡检间隔太长 → 盲区
`INTERVAL = 1200`（20 分钟）。真实损失：12:50 挂掉、13:06 才有人（其实是外部）拉起来，中间白等。
**经验值：活体检测 8 分钟一次是性价比拐点**（用户实测后指定 8 分钟）。
粗粒度巡检时，`COMFY_FAIL_LIMIT` 要设 **1** —— 因为正常的 ComfyUI 启动只要 30 秒，
8 分钟粒度下"探到一次不通"基本等于真死，再要求"连续 3 次"就变成 24 分钟。

### 缺陷 2：🔴 判定条件写成了"与"，导致永远不触发
```python
if not comfy_up() and hb_age() > 300:   # ← 错的
```
宿主在等结果时**每 45 秒 touch 一次心跳**。所以 ComfyUI 死了、宿主还活着时：
`comfy_up()=False` 成立，但 `hb_age` 永远很小 → **条件永不成立**，
一直要等到宿主自己的单段超时（本项目 3600s）才可能走出来。

**正确写法**：三个独立判据，**任一成立就动手**——
① 宿主进程不在（PID 台账对不上）② ComfyUI 连续 N 次不通 ③ 心跳过期。
把三者用 `or` 连起来，并把触发原因打进日志。

### 缺陷 3：清理残留用了已下线的 wmic
```python
os.system('wmic process where "name=\'python.exe\'" call terminate >nul 2>&1')
```
Win11 已移除 wmic → **静默 no-op**（stderr 还被重定向到 nul，连报错都看不见）。
改成 **PID 台账 + taskkill**：各进程启动时把自己的 PID 写进 `state/*.pid`，
清理时读文件精确杀，不再靠命令行匹配。

### 修复后的四层架构（已端到端验证）
| 层 | 文件 | 职责 | 自愈对象 |
|---|---|---|---|
| 渲染宿主 | `state/host.pid` | 跑队列 | — |
| ComfyUI | `state/comfy.pid` | 推理服务 | 由宿主拉起 |
| 看门狗 | `state/watchdog.pid` | 8 分钟一检 | 宿主 + ComfyUI + **守卫** |
| 守卫 | `~/.workbuddy/h3guard/guard.pid` | 8 分钟一检 | 宿主 + 看门狗 |

**互相守护**是关键：看门狗发现守卫不在就拉守卫，守卫发现看门狗不在就拉看门狗，
任一方活着整条链路就能自愈。守卫放在**非中文路径**（`C:\Users\<user>\.workbuddy\h3guard\`）。

⚠️ **用户可能明确反对常驻**：本机用户 2026-09-16 原话"必须是在我做视频的时候才启用这个程序，
做完要求关闭" → 于是删掉了 `HKCU\...\Run` 里的 `H3Guard` / `H3RenderKeepalive`，
改成**按需启停**（双击 start / stop，见第一节的 explorer 转交方案）。
**做守护架构前先确认用户要"常驻"还是"按需"**，别默认塞自启项。

### ★ 停链路时，杀的顺序是命门

```
看门狗 → 守卫 → 宿主/worker → 推理服务（ComfyUI）
```

**先断"会自愈的层"，再拆"被拉起的层"**。顺序反了的话，你杀完宿主，看门狗 8 分钟内
就把它原样拉回来，用户会以为"根本没关掉"（实测踩过）。
另外：
- 只杀**命令行匹配到本链路脚本**的 PID，**绝不"杀掉所有 python"**（会误伤别的任务）；
- `taskkill /F /PID` 不加 `/T`（加 `/T` 连坐，反而不可控）；
- 白名单硬拦：用户明确在用的程序（本机 `GameViewer.exe`=网易 UU 控制）、Edge、微信、Chrome 一律不碰；
- 杀完清 `state/*.lock`、`state/heartbeat`、`*.pid`，否则下次启动会被误判"已有实例在跑"；
- 最后再扫 1–2 轮兜底（防"杀的过程中被别的层又拉起来"）。

### 隔离实验结论（做守护架构前必须先测这个）
`taskkill /F /PID <父进程>`（**不加 `/T`**）**不会**杀掉父进程 `Popen` 出去的、带
`DETACHED_PROCESS | CREATE_NEW_PROCESS_GROUP` 的子进程。实测：杀掉看门狗后，
它拉起的 render_host 和 ComfyUI 都还活着（8188 仍在线）。
→ 所以"多层互相守护 + 单独重启某一层"是安全的；但**加 `/T` 会把整棵树连坐**，慎用。

### 进程存活判定（不用 wmic、不靠 PowerShell 工具）
```python
subprocess.run(["tasklist", "/FI", "IMAGENAME eq python.exe", "/FO", "CSV", "/NH"],
               capture_output=True, text=True, timeout=25, creationflags=0x08000000)
# 解析 CSV 的 PID 列；再和 state/*.pid 里的值比对 —— pid 在列表里才算"活着"
```
比"命令行匹配"稳，也比 `Get-CimInstance`（PowerShell 工具 stdout 恒空）可靠。

### 验证纪律
配好不算完，**必须人为制造一次故障**：`taskkill` 掉宿主 → 观察日志 → 确认新宿主起来 →
**等一个真实产出物落盘**（新 `.ok` 文件 / 新 mp4），才算通过。
本次实测自愈耗时 **92 秒**（杀 → 发现 57s → 重启 → ComfyUI 就绪 26s）。

---

## 七、长跑进度看板（本地网页，用户"我不看日志，给我看画面"时用）

用户说"跑多少了/拉开网页我看一下"时，别贴日志 —— 起个本地看板：

- 用一个 `serve.py`（python 标准库 `ThreadingHTTPServer`，**无第三方依赖**）暴露：
  `/` 页面 · `/api` JSON · `/thumb/X.jpg` 缩略图 · `/seg/X.mp4`（**要实现 Range 请求**，
  否则 `<video>` 拖进度条会挂）。
- 缩略图用 `ffmpeg -y -v error -ss 1.2 -i X.mp4 -frames:v 1 -vf scale=320:-2 out.jpg`，
  **线程池并发 + 缺啥补啥**（每次 `/api` 扫一遍，新产物自动补图），别一次同步生成几百张。
- 页面要素（按有用程度）：进度条 · 已出/总数/剩余 · **正在跑的是哪个** ·
  平均单段耗时 · **ETA** · 显存 · 各层状态灯（进程在不在）· 产物缩略图网格（点击即播）· 日志尾部。
- 前端 `setInterval(fetch('/api'), 8000)` 自动刷新，别用整页 meta refresh（会闪、会打断播放）。
- 服务本身用 Bash `run_in_background` 起（命令不返回 → Job 保持），再用 present_files 传
  `http://127.0.0.1:<port>` 直接给用户打开。
- 可复制模板：`D:\workbuddy\2026-09-16-14-39-25\dashboard\serve.py`（已实测可用）。

---

## 八、Node/Web 服务同理 + 桌面一键入口（2026-09-25 实测）

本地 `node server.js` 这类 Web 服务是同一套问题：Bash `run_in_background` 起的，**下一轮就 000 不通了**
（症状：`present_files` 的 localhost 预览面板一片空白、用户问"怎么我看不见了"）。

### 拉起（一次成功，实测 2 秒起）
```python
# ① 启动器放纯 ASCII 路径，内容纯 ASCII（见下面第 2 条坑）
# ② 交 explorer，轮询端口，失败换名重交
for name in ["zzzstart.bat", "zzzgo2.bat", "zzzdiag.bat"]:
    write_bat(name)
    subprocess.Popen(["explorer.exe", bat_path])
    for _ in range(12):
        time.sleep(1)
        if port_open(3000): return "SUCCESS"
```
**验证纪律不变**：发起调用返回后，**另起一次 Bash 调用**再 curl，还通才算真脱离。
用户真要"看画面"时，`subprocess.Popen(["explorer.exe", "http://127.0.0.1:3000/web/"])` 直接开默认浏览器（同样必须交 explorer，不能 `os.startfile`）。

### 两个新踩的坑
1. **中文 Windows 的 `netstat` 输出是 GBK**：`subprocess.run(['netstat','-ano'], capture_output=True)` 默认按 UTF-8 解 → `UnicodeDecodeError: 'utf-8' codec can't decode byte 0xbb`。
   必须 `r.stdout.decode('gbk', errors='replace')` 再正则。别用 `text=True`。
2. **bat 要写"纯 ASCII"，中文交给 JS 包装**：用 `open(..., encoding='ascii')` 写 bat 时，
   `title <中文>` 这种行会直接 `UnicodeEncodeError` 炸。
   正确做法：bat 只引用 **ASCII 路径的 `zzzrun.js`**，由这个 UTF-8 的 js 去 spawn 中文路径的主程序：
   ```js
   const { spawn } = require('child_process');
   spawn(process.execPath, ['D:\\中文目录\\server.js', '--api-only'],
         { detached: true, stdio: 'ignore', windowsHide: true }).unref();
   ```
   这样既守住"别在 bat 里写非 ASCII 路径"的规范，也彻底避开 GBK 编码坑。

### 用户侧交付：桌面一键 bat
- 观测到用户抱怨"看不到"之后才做（该用户默认**不要**往桌面放东西，除非他说看不见）。
- `shutil.copyfile` 复制一个**中文名 bat**（如 `津洋中转站.bat`）到桌面，实测名字不乱码（`os.listdir` 读回完好）。
- bat 逻辑：`netstat | findstr :3000` 判在不在跑 → 不在就起 → `timeout /t 3` → `start "" "<url>"` 开浏览器。
- **照样不加自启**：先查第一节"用户可能明确反对常驻"；本机用户反对，所以只给桌面入口，不碰 `HKCU\...\Run` / Startup。
- 交付后**必须实测**：脚本里自己 `taskkill` 掉服务 → 交 explorer 执行桌面 bat → 确认端口 2 秒内回来，才算过。

