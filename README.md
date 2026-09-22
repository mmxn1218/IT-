# IT工具箱 · 自动更新发布仓库

这个仓库**只放发布产物**，不放源码。

## 里面是什么

| 文件 | 作用 |
| --- | --- |
| `IT工具箱.dat` | 就是 `IT工具箱.exe`，只是改了后缀。**必须叫 `.dat`** —— jsDelivr 对 `.exe` 返回 403 |
| `version.json` | 版本描述，客户端启动时用它比对版本并校验下载 |

`version.json` 格式：

```json
{
  "version": "2026.09.22.1700",
  "md5": "791dcf3226c1b8db2b0010bbb6cfb4d2",
  "size": 115712
}
```

- `version` 格式 `YYYY.MM.DD.HHMM`，**必须和客户端编译进去的 `IT_VERSION` 完全一致**
- `md5` / `size` 必须是 `IT工具箱.dat` 的真实值，客户端下载后会逐一核对，不符就丢弃

## 客户端怎么更新

分两步。**第一步是为了绕开 jsDelivr 的 12 小时分支缓存**（见下一节）：

```
① 问「main 现在指向哪个 commit」
GET https://api.github.com/repos/mmxn1218/IT-/commits/main
  → {"sha":"24045d45…", …}

② 按 commit 取内容（commit 是不可变对象，永久缓存，取到就是新的）
GET https://fastly.jsdelivr.net/gh/mmxn1218/IT-@24045d45…/version.json
  → 版本号比本机新？
GET https://fastly.jsdelivr.net/gh/mmxn1218/IT-@24045d45…/IT工具箱.dat
  → 校验 size + md5
  → 换成新的 exe，重启
```

主地址不通时自动改走备用地址（同样钉在同一个 commit 上）：

```
https://gcore.jsdelivr.net/gh/mmxn1218/IT-@24045d45…/
```

真正走通的地址由客户端回传，**下载也用它**——否则会出现「检测到有新版本，但下不下来」。
两个地址都不通时，错误信息里会同时给出两个地址各自的失败原因。

第一步问不到 commit 时（限流 / API 不通 / 工厂内网只放行了 CDN），
**不会因此就不给更新**：自动退化成直接按 `@main` 取，只是可能要等缓存过期。

### ⚠️ 为什么要多问一次 commit —— 分支缓存 12 小时

jsDelivr 官方文档明确的规则：

| 寻址方式 | 缓存时长 |
| --- | --- |
| **分支**（`@main`） | **12 小时**，而且 **purge API 对分支无效** |
| **commit hash** | 基本永久（1 年头 + 永久 S3） |
| semver 版本别名 | 7 天（purge 只对这种有效） |

也就是说：**只按 `@main` 取的话，刚发的版本最长 12 小时之后客户端才看得到。**
2026-09-22 在本机实测，同一时刻：

```
@main/version.json          → 2026.09.22.1501      ← 缓存里的旧内容
@0a14e7283c7c/version.json  → 2026.09.22.1616      ← 已经是新的
@24045d45739f/version.json  → 2026.09.22.1616      ← 已经是新的
```

purge 清不掉分支缓存，等 12 小时又不现实 —— 所以只能加一条版本发现通道。
`api.github.com` 在本机实测 3/3 通、平均 219 ms，**公开仓库不需要任何凭据**
（匿名每小时 60 次，一台机器一天检查几次绰绰有余）。
顺带一提：`github.com:443` 和 `raw.githubusercontent.com:443` 在本机都不通，
反倒是 `api.github.com` 最稳。

**这个 12 小时缓存不是你这次才引入的问题** —— MES 生产日报那套更新机制里同样存在，
只是平时不容易被注意到（谁会在发完版之后 5 分钟就去另一台机器上验）。

想让客户端完全不碰 GitHub、就按 `@main` 走（接受 12 小时滞后），
把客户端 `IT工具箱.ini` 里的 `Api` 改成 `none` 即可：

```ini
[update]
Base=https://fastly.jsdelivr.net/gh/mmxn1218/IT-@main/
Fallback=https://gcore.jsdelivr.net/gh/mmxn1218/IT-@main/
Api=none
```

### ⚠️ 地址是**挑过的**，别随手改回 `cdn.jsdelivr.net`

2026-09-22 在本机（国内网络）实测，同一时刻四个入口的表现：

| 入口 | 结果 | 平均耗时 |
| --- | --- | --- |
| `cdn.jsdelivr.net` | **0 / 6，全被 RST（错误码 12031）** | ~3.0 s |
| `fastly.jsdelivr.net` | 5 / 5 通 | 0.14 s |
| `gcore.jsdelivr.net` | 5 / 5 通 | 0.12 s |
| `testingcf.jsdelivr.net` | 3 / 3 通 | 0.30 s |

`raw.githubusercontent.com` 在同一时刻是**完全不通**的。

原因不难理解：`cdn.jsdelivr.net` 是中文圈用得最多的那个域名，也最容易被 SNI 过滤；
冷门一点的入口反而更稳。**这四个是同一份内容的四个入口**（背后分别是 Fastly /
Gcore / Cloudflare / 自动），换地址不换内容，只是换条路。

所以默认值用的是「实测能通的」，不是「文档上最正统的」。工厂如果哪条更顺，
或者想指向自建内网镜像，直接在客户端「设置」里改（镜像场景记得把 `Api` 设成 `none`）。

## 怎么发新版本

1. 改源码里的 `IT_VERSION`（`update.h`），格式 `YYYY.MM.DD.HHMM`
2. 编译出新的 `IT工具箱.exe`
3. 发版（自动：改名成 `.dat` → 算 md5/size → 写 `version.json` → commit → push
   → **问 commit → 按 commit 回读 version.json → 实际下载 `.dat` 核对 md5**）

```bat
IT工具箱_发布助手.exe --yes --no-proxy
```

> 发版顺序不能错：**先改版本号再编译**。版本号没变的话，客户端会认为已经是最新，不会下载。
> 发布助手会拿 `update.h` 里的版本号去 exe 里搜一遍，**搜不到就直接拒绝发布**——
> 防的就是「改了版本号忘了重新编译」。

核对那一步**不需要任何等待**：以前的实现是「purge + 睡 8 秒再回读」，
而按官方规则 purge 对分支文件根本无效，所以那一步在这条地址上从来就不可能成功。
现在改成照客户端的真实路径走，取到即生效。

核对完还会顺便报一句 `@main` 现在给的是什么版本 —— 这是**分支缓存的现场证据**，
留着是为了下次发版时不用再怀疑「是不是我没推上去」。

### github.com 被墙时怎么发

`git push` 走的是 `github.com:443`；国内这个地址经常直接不通。
而 GitHub 的 **REST API 在 `api.github.com`，通常还是通的**。所以分三步走：

```bat
:: 1) 生成产物 + 本地提交，不 push
IT工具箱_发布助手.exe --yes --no-proxy --no-push

:: 2) 走 api.github.com 把这次提交推上去（等价于 git push）
python tools\push_via_api.py --repo "D:\wbuddy\IT工具箱_发布" --msg "发布 IT工具箱"

:: 3) 核对远端真的更新了
IT工具箱_发布助手.exe --verify-only
```

`push_via_api.py` 用 Git Data API（blob → tree → commit → 更新 ref）造出一个
和 `git push` 效果完全一样的提交；凭据从 Windows 凭据管理器取，**只在内存里用**。

## 与 MES 生产日报的两处不同（都有意为之）

**一、替换方式不同。**
MES 那边是**写一个 bat 自覆盖重启**。这里换成了**自我复制式 exe 更新器**：

主程序把自己复制成 `_it_updater.exe`，用 `--apply-update <pid> <exe>` 命令行驱动它
等旧进程退出 → 换文件 → 重启。失败还能回滚。

理由是 bat 这条路**没法被自动测到**：bat 得存成 GBK、而且执行它要拉起 `cmd.exe`，
在受限环境里会被拦，出了事只能靠用户肉眼发现。换成 exe 之后，「替换」「回滚」
「重启」每一步都能被自检断言覆盖。

**二、版本发现多一条通道。**
MES 只按 `@main` 取。这里加了 `api.github.com` 拿 commit，
做到「发版即生效」，而不是「最长 12 小时后生效」。理由见上面那节。

## 客户端自检

`IT工具箱_exe` 那边有 **335 项自检**，发版前应当全绿：

| 套件 | 项数 | 覆盖 |
| --- | --- | --- |
| 数据层 | 63 | 清单增删改查、默认 14 条、网址规范化、启动程序（含真启动探针） |
| 界面 | 56 | 跨进程驱动真实窗口：顶栏布局不重叠、卡片筛选/搜索、点卡片真启动程序、两个弹窗 |
| 更新层 | 216 | 纯逻辑 + 真下载 + 重试 + **按 commit 寻址** + 替换重启 + 端到端；**自带本地 HTTP 假更新源，不联网、不依赖 Python** |

还有一个小工具，排查「这个点位能不能更新」很好用（逐次报耗时和错误码，
一眼能分层看出是 DNS / TCP / TLS / 被 RST 还是 CDN 缓存）：

```bat
selftest\_netcheck.exe "https://fastly.jsdelivr.net/gh/mmxn1218/IT-@main/version.json" 8
selftest\_netcheck.exe --dump "https://api.github.com/repos/mmxn1218/IT-/commits/main"
```

`--dump` 会把响应体前 600 字节也打出来 —— **「通了」不等于「拿到的是新东西」**，
CDN 缓存问题只有看内容才判断得了。注意本机用 `curl` 会被沙箱/代理挡成假 200，
这类探测必须用原生进程走真网络。

### 几条值得单独记住的断言

- **顶栏四件套不许重叠**：加按钮时把搜索框挤重叠过，所以有断言守着。
- **弹窗里的控件必须落在客户区内**：布局里写 `y += SC(20)`（逻辑坐标加物理像素）
  在 96 DPI 下完全等价、看不出来；到 225% 缩放时下半截控件会被推出窗口，
  「保存 / 取消」直接看不见，而「按 ID 找控件 + BM_CLICK 点按钮」的测试照样全绿。
  只有比较「控件矩形 vs 客户区矩形」才抓得住。
- **连接被重置要能靠重试扛过去**：自检里内置服务会故意把前 N 次连接 RST 掉，
  断言客户端最终仍能拿到内容；同时断言「抖满上限时老实失败并报出原因」，
  以及「总耗时不超预算」（否则开机静默检查会硬卡）。
- **12 小时分支缓存必须有离线复现**：自检里造出
  `_t\gh\o\r@main\`（旧内容）和 `_t\gh\o\r@<sha>\`（新内容）两份 version.json，
  断言「不解析 commit 就会误判已是最新」「解析 commit 就立刻看到新版」。
  这个坑不写成断言，下次重构很容易又踩回去。
- **配置地址一个字节都不许被改写**：`upd_check4` 回传的 `used_base` 是钉过 commit 的，
  只能用于本次下载。要是被写回 ini，这台机器就永久停在那个版本、再也收不到更新 ——
  而且**手动点一次「检查更新」就会触发**。所以既有「传进来的 base 不被改动」的断言，
  也有客户端侧把「配置地址」和「本次生效地址」分成两个变量。
- **默认地址 / 文件名不许在测试里抄第二遍**：全部从 `update.h` 现推。
  抄过两次，两次都因为改默认值而假红/假绿。

## 注意

- 仓库必须是 **public**，否则 raw / jsDelivr 取不到文件
- `api.github.com` 匿名每小时 60 次。一台机器一天检查几次完全够；
  要是哪天不够用了，说明有机器在疯狂重启，该查的是那个
- `.gitattributes` 里把 `*.dat` / `*.exe` 标成 `binary`、`*.json` 标成 `-text`，
  **不要删**：否则换行转换会改掉字节，md5 和 `version.json` 里的就对不上了
- `.gitignore` 也别删：发布助手和客户端会在工作目录里留下临时文件
  （`_it_*.bin` / `_it_old.exe` / 核对用的下载文件），`git add -A` 会把它们一起提交
