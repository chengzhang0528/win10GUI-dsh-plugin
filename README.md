# DeskPilot for DeepSeek Harness

把 **Windows 10 交互式桌面**交给 DSH 的 Agent：窗口/控件观察、UIA 动作、真实键鼠、可信截图、离线 OCR，以及 Chrome 的 CDP 页面操作。

一句话价值：装上之后，Agent 能在你的桌面上**真的动手**——看到窗口、点到按钮、填好表单、读出结果，而不是只给建议。

- 宿主平面插件：提供 `ctx.deskpilot`，**每个会话一个常驻 `win-agent.exe` NDJSON 进程**
- 4 个模型工具：`deskpilot_doctor` / `deskpilot_run` / `deskpilot_batch` / `deskpilot_session`
- 2 条斜杠命令：`/deskpilot`、`/dp`（不消耗模型回合）
- 6 个 Agent Skills：`deskpilot-core`、`deskpilot-browser`、`deskpilot-messaging`、`deskpilot-wechat-assistant`、`deskpilot-testing`、`deskpilot-flow-evolution`

## 环境要求

| 项 | 要求 |
|---|---|
| 系统 | **Windows 10 build 19041+，x64**（其他平台不支持） |
| 桌面 | 已登录、可交互的桌面会话（锁屏/服务会话不在承诺范围） |
| 宿主 | DeepSeek Harness（`dsh`）+ `pnpm` |
| 运行时 | 首次需要 .NET Desktop Runtime 10 x64；**原生入口会自动下载并安装**，可能弹一次 Windows UAC |

## 安装

三步：装插件 → 启用并指向制品 → 重启宿主。

### 1. 装插件包

从本仓库的最新 Release 资产安装（固定不可变）：

```powershell
dsh plugin --profile web add "C:\path\to\dsh-plugin-deskpilot-0.2.0.tgz"
```

`dsh plugin` 会在 profile 目录里调用 pnpm，并把本包并入 `dsh.profile.bundles`。**不需要 npm 账号或 `npm login`**，包也不来自 npm Registry。

> 也可以直接从源码目录安装：`dsh plugin --profile web add "D:\path\to\win10GUI-dsh-plugin"`。

### 2. 启用插件行并指向 CLI 制品

插件只贡献一个**默认禁用**的行——安装不等于授权它驱动你的桌面。打开 profile 自己的用户层 `cordis.patch.yml`（**不要**改 dsh 随附的任何 preset，也不要整份覆盖该文件，里面可能有你自己的配置），加入：

```yaml
- id: deskpilot
  disabled: false
  config:
    # CLI 制品：固定不可变地址 + 字节数 + SHA-256
    asset:
      version: '6947fa5'
      platform: win-x64
      url: 'https://shared-public-assets.oss-cn-beijing.aliyuncs.com/deskpilot/cli/releases/6947fa5/dsh-bootstrap-6947fa5.zip'
      bytes: 13918926
      sha256: '6f0a3949ca155bb5106bbe2f18e6cb777e8c8ff2c1eebeb063e662585a1c21ba'
    timeoutMs: 130000
    doctorTimeoutMs: 60000
    setupTimeoutMs: 900000
    maxLineBytes: 8388608
    registerSkills: true
```

**不要**把 `command` 指向便携包内部的 `bin\app\win-agent.exe`：那会绕过运行时自举检查。插件会直接拒绝这种配置并告诉你正确的公开入口。

### 3. 重启宿主并自证

退出并重新打开 dsh 宿主（bundles 层在启动时组合），在新会话调用 `deskpilot_doctor`。

**完成标准**：`response.ok == true`。首次启动可能下载并准备 .NET Desktop Runtime，也可能出现 UAC，这属于正常首次准备，会占用 `setupTimeoutMs`（默认 15 分钟）而不是普通请求预算。

## 权限与风险（安装前请读）

- **它会操作你的真实桌面**：模拟点击、键盘输入、前台窗口切换，并使用你当前的 Chrome profile。请在有授权的前提下使用。
- **首次会修改系统运行时**：下载微软官方 .NET Desktop Runtime 安装包，校验 SHA-512 与 Authenticode 签名后静默安装，必要时请求 UAC 提权。
- **不绕过安全边界**：不绕过 UAC、安全桌面、密码控件；登录、验证码、OTP 一律留给用户。
- **不持久化敏感内容**：不记录密钥、输入正文或客户数据；截图是否保留由调用方决定。
- **网络**：仅从上面配置的固定 OSS 地址下载 CLI 制品，以及从微软官方地址下载运行时；不会去 GitHub 或 npm Registry 找替代源。

## 工具与命令

| 名称 | 作用 | 约束 |
|---|---|---|
| `deskpilot_doctor` | 报告本机能否被自动化（UIA、SendInput、可信截图、OCR、Chrome/CDP、DPI） | 一次性进程，不占驱动会话 |
| `deskpilot_run` | 发一条**只读**请求，原样返回 CLI 结构化响应 | 方法白名单；变更类会被拒绝并提示改用批处理 |
| `deskpilot_batch` | 一次 `actions.batch`/`workflow.run`，最多 32 步 | 必须给 `reason`；首个错误即停，不是事务 |
| `deskpilot_session` | 查看/重置驱动会话 | 只动会话，不关 Chrome |
| `/deskpilot`、`/dp` | `status` / `doctor` / `windows` / `reset confirm` / `setup` | 直接跑在同一会话上，不进模型历史 |

## 配置项

| 字段 | 默认 | 说明 |
|---|---|---|
| `asset` | 无 | `{version, platform:'win-x64', url, bytes, sha256}`；未配置且无本地 `command` 时报明确缺失，**不尝试任何回退源** |
| `command` | 无 | 已有便携包时直接指定公开入口绝对路径（可离线） |
| `cacheRoot` | `$DSH_HOME/cache/deskpilot` | 制品解包缓存根 |
| `timeoutMs` | `130000` | 单次请求预算 |
| `doctorTimeoutMs` | `60000` | `doctor` 预算（未进入准备流程时） |
| `setupTimeoutMs` | `900000` | 仅收到原生入口的准备信号后才启用 |
| `maxLineBytes` | `8388608` | 单行响应上限 |
| `skillRoots` | 无 | 覆盖技能根；留空表示不贡献 |
| `registerSkills` / `registerCommands` | `true` | 可分别关闭技能与斜杠命令 |
| `startupDiagnostic` | 无 | 给绝对路径则写出启动自检 JSON（工具/技能/命令的实际注册结果） |

## 制品与校验

| 制品 | 地址 | 字节 | SHA-256 |
|---|---|---|---|
| 插件 TGZ 0.2.0 | `…/deskpilot/dsh-plugin/releases/0.2.0/dsh-plugin-deskpilot-0.2.0.tgz` | 26251 | `9fb23163012b872431c09880fbe8c54b94e8ef1d09d330c2190d8e9226758fe7` |
| CLI bootstrap 6947fa5 | `…/deskpilot/cli/releases/6947fa5/dsh-bootstrap-6947fa5.zip` | 13918926 | `6f0a3949ca155bb5106bbe2f18e6cb777e8c8ff2c1eebeb063e662585a1c21ba` |

插件对 CLI 制品的校验不止于 SHA-256：解包后会按便携包 `PACKAGE_MANIFEST.json` **逐文件比对**大小与摘要，拒绝符号链接、路径穿越、未声明文件和缺件（含许可证与 Skills）。只有全部通过才会激活缓存。

## 故障排查

| 现象 | 处理 |
|---|---|
| `DESKPILOT_RUNTIME_SETUP_FAILED` | 首次运行时准备失败（UAC 被拒、网络、安装器报错）。**先看错误里的 stderr 尾部**；不要自动重放业务写操作。修复后重新发起一次请求。 |
| `DESKPILOT_RUNTIME_SETUP_TIMEOUT` | 准备超时。给足时间后重试；必要时手工安装 .NET Desktop Runtime 10 x64。 |
| `DESKPILOT_ASSET_INVALID` | 制品清单缺件/多件/含链接，或 `asset` 字段不合法（platform 必须 `win-x64`，bytes ≤ 512 MiB，sha256 为 64 位十六进制）。 |
| `DESKPILOT_ASSET_INTEGRITY` | 下载字节数或 SHA-256 不符。不要绕过校验，检查代理/镜像是否篡改。 |
| `DESKPILOT_ASSET_DOWNLOAD_FAILED` | 制品地址不可达或返回非 2xx；插件不会回退到其他来源。 |
| `DESKPILOT_INTERNAL_ENTRYPOINT` | `command` 指向了 `bin\app\win-agent.exe`，改用公开入口 `bin\win-agent.exe`。 |
| `DESKPILOT_TIMEOUT` / `DESKPILOT_NO_RESPONSE` | 普通请求超时/进程退出无响应；看 `deskpilot_session` 的 `last_exit`，重新观察后继续。 |

## 兼容性

- 目标宿主：DSH `0.1.6-alpha.1`（peer：`@deepseek-ai/cordis` ≥3、`@deepseek-ai/dsh-tools` ≥0.1.6-alpha.1、`@deepseek-ai/schemastery` ≥3）。
- 插件为**宿主平面** bundle：不修改 dsh 随附 preset，不提供客户端 UI。
- 平台：仅 `win-x64`。非 Windows 宿主不会加载该行。

## 修改后验证

```powershell
npm --prefix . test           # 单元测试
npm pack --dry-run            # 确认发布清单：仅源码/补丁/README/LICENSE，无二进制
```

真实候选的宿主检查见 `test/host-smoke.mjs`（需显式指定候选公开入口与宿主安装目录；只跑能力与 doctor，不操作现有桌面应用）。

## 许可

插件本身 MIT（见 `LICENSE`）。CLI 与第三方组件许可见制品内的 `bin/app/licenses/`。

## 相关

- CLI 与 Skills 源码：[chengzhang0528/window10GUI](https://github.com/chengzhang0528/window10GUI)
- 社区插件目录：[dsh-plugin.org](https://dsh-plugin.org)
