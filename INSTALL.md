# dsh-office-tools — DSH Desktop 安装说明（AI / 自动化可执行）

> 本说明**仅针对 DSH Desktop 桌面版**（Electron 桌面应用，harness profile 为 `web`）。本插件为 **host-only**（无客户端 UI），安装后必须**重启桌面应用**才生效。

## 0. 前置条件

- 已安装并运行 **DSH Desktop**。
- 已取得成品包 `dsh-office-tools-0.5.1.tgz`（来源：GitHub Releases 下载，或源码 `pnpm run build` + `npm pack` 构建产物）。

> 成品包为**自包含**：`lib/index.js` 已内联全部 Office 库（docx / xlsx@0.20.3 / pptxgenjs / jszip），运行时不需要解析任何 Office 依赖、也不需要访问 SheetJS CDN；仅 `@deepseek-ai/*` 与 cordis 由 DSH harness 提供。

## 1. 定位 `dsh` 命令行

DSH Desktop 不保证 `dsh` 在系统 PATH 上。先探测：

```powershell
dsh --version
```

- 若打印版本号 → 直接进入第 2 步，命令用 `dsh ...`。
- 若报「无法识别 dsh」→ 改用桌面应用捆绑的 node + dsh bin，先定义变量（按实际安装目录替换 `<INSTALL_DIR>`）：

```powershell
$node = "<INSTALL_DIR>\DSH Desktop\resources\app\node_modules\node\bin\node.exe"
$dsh  = "<INSTALL_DIR>\DSH Desktop\resources\app\node_modules\@deepseek-ai\dsh\lib\bin.js"
```

> 本机示例：`<INSTALL_DIR>` = `G:\desktop-dsh`。

## 2. 安装插件

把 `<TARBALL>` 替换为 tgz 的绝对路径。

```powershell
dsh plugin --profile web add "<TARBALL>\dsh-office-tools-0.5.1.tgz"
```

若 `dsh` 不在 PATH，用：

```powershell
& $node $dsh plugin --profile web add "<TARBALL>\dsh-office-tools-0.5.1.tgz"
```

## 3. 校验安装结果

安装成功应满足全部 3 条（Windows 下 profile 目录为 `%APPDATA%\dsh-desktop\harness\profiles\web`）：

1. `profiles\web\package.json` 的 `dependencies` 含 `"dsh-office-tools"`；
2. 同一文件 `dsh.profile.bundles` 数组含 `"dsh-office-tools"`；
3. `profiles\web\node_modules\dsh-office-tools\lib\index.js` 存在（自包含 host 产物）。

## 4. 重启 DSH Desktop（必做，不可省略）

- 完全退出：系统托盘 → DSH Desktop 图标 → 右键 → **退出**；
- 重新打开 DSH Desktop。

（重启后 harness 才会加载插件的 host 半；不重启插件不生效。）

## 5. 验证插件已加载

host 插件无客户端 UI，验证方式：

1. 打开任意会话，对模型说「用 word_create 在当前工作区创建一个测试 docx」；
2. 模型应能调用 `word_create` 并生成 `.docx` 文件；
3. 或查看模型可用的工具清单，应包含 8 个工具：

| 工具 | 作用 |
|---|---|
| `word_create` / `word_read` / `word_update` | Word `.docx` 创建 / 读取 / 追加更新 |
| `excel_create` / `excel_read` / `excel_update` | Excel `.xlsx` 创建 / 读取 / 更新（`=` 开头字符串写成真实公式） |
| `ppt_create` / `ppt_read` | PowerPoint `.pptx` 创建（支持 PNG/JPG/GIF 图片）/ 读取 |

## 6. 卸载 / 回滚

```powershell
dsh plugin --profile web remove dsh-office-tools
```

（或手动：从 `profiles\web\package.json` 移除 `dsh-office-tools` 依赖与 `bundles` 条目，然后重启 DSH Desktop。）

---

## 附：可选配置 `enablePptTools`

默认 `true`（注册 `ppt_create` / `ppt_read`）。若同时使用 dsh-ppt 等专用 PPT 插件（会注册同名 `ppt_create`，DSH 启动时拒绝重名工具），可关闭本插件的 PPT 对：

```yaml
# profile 的 cordis.patch.yml（注意：不要重复 insert 同 id）
- insert:
    - id: dsh-office-tools
      config:
        enablePptTools: false
```

## 附：安全边界（模型使用本插件时）

- 所有读写限制在发起调用的 agent 的**会话工作目录**内（相对路径按 `session.header.cwd` 解析，绝对路径需位于工作目录内，realpath 防符号链接逃逸）。
- 读取上限 50 MiB；zip 炸弹防护（单条目 ≤256 MiB、整包 ≤512 MiB、条目 ≤100 000，超限直接拒绝）；带 DOCTYPE/ENTITY 声明的 XML 部件拒绝。
- 创建/更新默认 `overwrite: false` 防误覆盖；文本/单元格结果有上限并标记 `truncated`。
- 不调用 LibreOffice / Word / Excel 等外部进程，纯 JS 库生成/解析。
