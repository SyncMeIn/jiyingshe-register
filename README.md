# 集英社 JumpShop 会员批量自动注册脚本项目介绍

> **项目名**：`jiyingshe-register`  
> **定位**：面向集英社 JumpShop 会员注册流程的自动化批量注册工具。  
> **语言**：Python 3.11+  
> **主要依赖**：Playwright、BitBrowser-bridge、xlrd、python-dotenv、loguru、requests。

---

## 一、项目背景与要解决什么问题

集英社 JumpShop 的会员注册流程较为繁琐：

- 需要填写邮箱、密码、确认密码；
- 需要接收邮件验证码并回填；
- 需要填写姓名、平假名、邮编、地址、性别、生日等 14 项个人信息；
- 注册成功后还需要日本手机号进行 SMS 二次验证；
- 频繁操作对 IP、浏览器指纹、操作节奏都有风控要求。

`jiyingshe-register` 的目标就是把这些重复步骤自动化：从 Excel 读取待注册账号，使用浏览器自动化完成注册、收邮件、填表、接 SMS 验证码，并把结果写入 CSV。适合需要批量注册、对成功率与可追溯性有要求的场景。

---

## 二、核心功能亮点

| 亮点　　　　　　　　　 | 说明　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　|
| ------------------------| -------------------------------------------------------------------------------------|
| **Excel 批量输入**　　 | 账号信息来自 `docs/7.100.xls`，一行一个账号，支持从指定行起读、设置处理数量。　　　 |
| **邮件验证码自动拉取** | 通过 IMAP 连接 iCloud 邮箱，按发件人 `shueisha` 匹配最近邮件，正则提取 6 位验证码。 |
| **SMS 接码集成**　　　 | 接入火狐狸（firefox.fun）接码平台，自动获取日本手机号、接收 SMS 并回填。　　　　　　|
| **指纹浏览器**　　　　 | 使用 `BitBrowser` 启动每次独立的浏览器指纹，降低被检测风险。　　　　　　　　　　　|
| **代理支持**　　　　　 | 支持带用户名密码的 HTTP 代理，适配日本线路需求。　　　　　　　　　　　　　　　　　　|
| **断点续跑**　　　　　 | 通过 `.last_start_row` 记录上次处理到的行号，下次启动自动从断点继续。　　　　　　　 |
| **完整闭环验证**　　　 | 注册成功后退出并重新登录，验证账号真正可用。　　　　　　　　　　　　　　　　　　　　|
| **日志 + CSV 结果**　　| 使用 `loguru` 记录控制台与日志文件，每行结果写入 `result.csv`。　　　　　　　　　　 |
| **可打包为 EXE**　　　 | 提供 `run.py` 作为 PyInstaller 入口，支持打包成独立可执行文件。　　　　　　　　　　 |

---

## 三、整体流程（5 步闭环）

完整流程在 `src/registrar.py` 中实现：

1. **Step 1：发送邮件验证码**  
   访问注册页，填写邮箱、密码、确认密码，点击「送信する」，进入个人信息页。

2. **Step 2：填写个人信息与验证码**  
   自动填写姓名、平假名、邮编、地址、性别、生日，并勾选隐藏条款；通过 IMAP 拉取 6 位验证码回填。

3. **Step 3：最终确认注册**  
   在确认页点击「登録する」，进入 SMS 验证页即表示提交成功。

4. **Step 4：SMS 接码验证**  
   调用火狐狸获取日本手机号，填入 `auth_tel`，等待并回填 SMS 验证码完成认证。

5. **Step 5：登录闭环验证**  
   退出当前会话，使用邮箱 + 密码重新登录，确认账号可正常进入会员中心（マイページ）。

`src/main.py` 负责读取账号、循环调用注册流程、记录结果、保存断点行号。

---

## 四、主要技术模块

| 文件　　　　　　　　　　　 | 职责　　　　　　　　　　　　　　　　　　　　　　　 |
| ----------------------------| ----------------------------------------------------|
| `src/main.py`　　　　　　　| 程序入口，解析命令行参数，驱动主循环。　　　　　　 |
| `src/registrar.py`　　　　 | 注册核心：5 步流程 + 重试 + 错误处理。　　　　　　 |
| `src/browser.py`　　　　　 | 启动 `BitBrowser` 指纹浏览器，处理弹窗。　　　　 |
| `src/config.py`　　　　　　| 从 `.env` 读取代理、邮箱、接码平台、浏览器等配置。 |
| `src/excel_reader.py`　　　| 读取 `.xls` 账号数据，映射为 `Account` 对象。　　　|
| `src/mail_client.py`　　　 | IMAP 客户端，轮询并提取邮件验证码。　　　　　　　　|
| `src/phone_client.py`　　　| 火狐狸接码平台客户端。　　　　　　　　　　　　　　 |
| `src/checkpoint.py`　　　　| 持久化起始行号，支持断点续跑。　　　　　　　　　　 |
| `scripts/launch_chrome.sh` | 开发调试时启动带 CDP 的本地 Chrome。　　　　　　　 |
| `run.py`　　　　　　　　　 | PyInstaller 打包入口，运行时自动切换工作目录。　　 |

配置示例在 `.env.example` 中，复制为 `.env` 后填入真实信息即可运行。

---

## 五、项目演进（从提交记录看成长路线）

根据 Git 提交记录，项目经历了几次关键迭代：

1. **初始化项目** — 搭建最小可运行骨架，验证注册页面可访问。  
2. **完整注册流程** — 实现 Step 1~3，完成邮件验证码与个人信息填写。  
3. **集成火狐狸接码 + YesCaptcha 打码** — 解决 SMS 验证与 reCAPTCHA 自动处理。  
4. **优化注册流程与重试机制** — 增强错误检测、字段重试、自动回退。  
5. **启动脚本 + 防旧码复用** — 增加 `bat` 启动文件，优化 SMS 轮询避免重复旧码。  
6. **指纹浏览器 + 文档优化** — 引入 BitBrowser，降低风控风险；同步更新 README 安装说明。  
7. **PyInstaller 打包 + 登录流程优化** — 支持打包 EXE，完善注册后登录验证闭环。  

可以看出，项目从「能跑」逐步走向「稳定、可交付、可分发」。

---

## 六、快速上手

```bash

# 1. 安装依赖
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
PLAYWRIGHT_DOWNLOAD_HOST=https://npmmirror.com/mirrors/playwright \
  playwright install chrome

# 2. 复制并编辑配置
cp .env.example .env
# 填写 STATIC_PROXY_URL、MAIL_USER、MAIL_PASS、FIREFOX_TOKEN 等

# 3. 运行（默认从 Excel 第 3 行开始，处理 1 条）
python3 -m src.main --limit 1

# 4. 批量运行（例如从第 3 行开始，处理 5 条）
python3 -m src.main --file docs/7.100.xls --start-row 3 --limit 5

```

结果会写入 `result.csv`，日志按天写入 `logs/`。

---

## 七、适用场景与使用建议

- 适合需要批量、重复完成集英社 JumpShop 会员注册的场景；
- 建议在运行前确认代理稳定、iCloud 应用专用密码有效、火狐狸余额充足；
- 建议先用 `--limit 1` 做单条验证，再逐步放量；
- 如遇到风控，可调整 BitBrowser 的 `geoip`、`humanize` 参数或更换代理 IP。

---

## 八、未来可拓展方向

- 并发多账号处理，提升整体吞吐；
- 图形验证码自动识别（若后续出现）；
- 失败队列与重试策略，进一步提高成功率；
- Docker 化部署，便于服务器端长期运行；
- 增加 Web 可视化面板，实时查看任务进度与成功/失败统计。

---

## 结语

`jiyingshe-register` 是一个把「浏览器自动化 + 邮件验证码 + SMS 接码 + 指纹浏览器 + 断点续跑」组合起来的典型自动化工具。它不仅解决了集英社 JumpShop 会员注册流程复杂、重复度高的问题，也通过模块化设计让后续维护和功能扩展都相对清晰。如果你也在做类似的海外平台批量注册或自动化任务，欢迎参考项目结构与实现思路。

> 注：本项目代码仅供技术学习与交流，使用请遵守目标平台服务条款与相关法律法规。


## 部分截屏

![](https://s.gqmg.com/https://oray.kainy.cn:38400/_upload/./content/temp/2026/07/mrdddhx7.webp)



![](https://s.gqmg.com/https://oray.kainy.cn:38400/_upload/./content/temp/2026/07/mrddegv3.webp)



![](https://s.gqmg.com/https://oray.kainy.cn:38400/_upload/./content/temp/2026/07/mrddgz3e.webp)

## 启动信息

```bash
=========================================
  集英社 JumpShop 自动注册系统
=========================================
[+] Python 已安装
[+] 虚拟环境已存在
[+] 检查/安装依赖...

[notice] A new release of pip is available: 25.0.1 -> 26.1.2
[notice] To update, run: python.exe -m pip install --upgrade pip
[+] 已切换至 CloakBrowser 指纹浏览器

请输入起始行 (默认 35):

请输入处理条数 (默认 268): 

失败时是否保持浏览器打开? (y/N, 默认 y):

```

## 执行日志

```bash

18:33:33 | INFO    | [main] loaded 268 accounts from docs/7.100.xls (start_row=35)
18:33:33 | INFO    | ===== [1/268] referee.menorah-1b@icloud.com (row=35) =====
18:33:33 | INFO    | [browser] launching CloakBrowser (headless=False, geoip=True, humanize=True)
18:33:35 | INFO    | [firefox] 余额: 163.4800
18:33:35 | INFO    | [firefox] getPhone iid=2850 country=jpn
18:33:36 | SUCCESS | [firefox] phone=07095163299查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单, pkey=3B3F9AB2E76B...
18:33:35 | INFO    | [firefox] getPhone iid=2850 country=jpn
18:33:35 | INFO    | [firefox] getPhone iid=2850 country=jpn
18:33:36 | SUCCESS | [firefox] phone=07095163299查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单, pkey=3B3F9AB2E76B...
18:33:36 | SUCCESS | [firefox] 手机号: 07095163299查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单
18:33:36 | INFO    | [step1] 清除 cookies 确保新会话
18:33:36 | INFO    | [step1] goto https://jumpcs.shueisha.co.jp/shop/customer/entryonetimepasswordsend.aspx
18:33:42 | INFO    | [step1] mail=referee.menorah-1b@icloud.com pwd=Oneaaa8008        
18:34:04 | INFO    | [step1] 点击 送信する (attempt=1)
18:34:12 | INFO    | [step1] ✅ → https://jumpcs.shueisha.co.jp/shop/customer/entry.asp?username=fdb9a45d8e84d0dd5cbb23e2aecdc91630d75b56cae55fe96208cf1651220410cfe3216ceddx?username=fdb9a45d8e84d0dd5cbb23e2aecdc91630d75b56cae55fe96208cf1651220410cfe3216ceddedd5d0c2950f7d4b797a09df5264912e5bd97045c5e94a2c55509
18:34:12 | INFO    | [mail] 等待验证码邮件 (referee.menorah-1b@icloud.com)
18:34:12 | INFO    | [mail] connecting imap.mail.me.com:993
18:34:13 | INFO    | [mail] connected (2451 msgs)
18:34:28 | INFO    | [mail] #1 轮询中... (等14s, seq≤2451)
18:34:31 | SUCCESS | [mail] #2 ✅ 验证码=411781 | 收件=referee.menorah-1b@icloud.com | eq=2452 | Thu, 9 Jul 2026 10:34:10 +0000
seq=2452 | Thu, 9 Jul 2026 10:34:10 +0000
18:34:32 | SUCCESS | [mail] 验证码: 411781
18:34:32 | INFO    | [step2] 验证码=411781 密码=Oneaaa8008
18:35:23 | INFO    | [step2] 点击 確認画面へ (attempt=1)
18:35:29 | INFO    | [step2] ✅ → https://jumpcs.shueisha.co.jp/shop/customer/entry.asp
x
18:35:31 | INFO    | [step3] 点击 登録する
18:35:40 | INFO    | [step3] ✅ 进入 SMS 页
18:35:40 | INFO    | [step4a] 填 SMS 电话: 07095163299查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单
18:35:48 | INFO    | [step4a] SMS 已发送到 070****3299
18:35:48 | INFO    | [step4b] 等待火狐狸 SMS 验证码 (pkey=3B3F9AB2E76B...)
18:36:01 | SUCCESS | [firefox] #9 ✅ SMS=491925 (12s)
18:36:04 | INFO    | [step4b] 填入验证码: 491925
18:36:10 | INFO    | [step4b] SMS验证完成，URL: https://jumpcs.shueisha.co.jp/shop/customer/menu.aspx?authKey=
18:36:10 | INFO    | [step5] 已在登录态（step4 SMS 后），验证后开始登出
18:36:10 | INFO    | [step5] 点击 ログアウト 退出...
18:36:15 | INFO    | [step5] 点击 ログイン 链接
18:36:34 | INFO    | [step5] 点击 ログイン 按钮
18:37:11 | SUCCESS | [step5] ✅ 登录成功！
18:37:12 | WARNING | [firefox] 释放失败: 0|-4
18:37:12 | INFO    | ===== [2/268] 93jowly-stria@icloud.com (row=36) =====
18:37:12 | INFO    | [browser] launching CloakBrowser (headless=False, geoip=True, humanize=True)
18:37:14 | INFO    | [firefox] 余额: 161.4400
18:37:14 | INFO    | [firefox] getPhone iid=2850 country=jpn
18:37:15 | SUCCESS | [firefox] phone=07093492410查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单, pkey=3B3F9AB2E76B...
18:37:15 | SUCCESS | [firefox] 手机号: 07093492410查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单查运单
18:37:15 | INFO    | [step1] 清除 cookies 确保新会话
18:37:15 | INFO    | [step1] goto https://jumpcs.shueisha.co.jp/shop/customer/entryonetimepasswordsend.aspx
18:37:21 | INFO    | [step1] mail=93jowly-stria@icloud.com pwd=Oneaaa8008

```


注意，本项目仅在 SyncMein 的500人小群分享。

加微进群，谢谢你看完了我的文章，我们下次再见吧。

> / 作者：明察
