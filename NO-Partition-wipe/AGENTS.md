# 💻 PowerShell 执行规范

在执行任何脚本、命令或处理环境交互时，所有 CLI 命令和脚本执行必须严格遵守以下 PowerShell 规则：

1. **统一 Shell 环境**：所有命令必须使用 PowerShell (推荐 `pwsh` 即 PowerShell 7 以上)。禁止使用 Windows 旧版 `cmd.exe` 或默认 `powershell.exe`。
2. **转义字符约束**：Bash 与 PowerShell 的转义规则不同，在编写跨平台脚本时，禁止混用 Shell 语法。
3. **严格的错误处理**：所有脚本首部必须包含 `$ErrorActionPreference = 'Stop'`，确保遇到错误时立即中断并报错，避免静默失败。
4. **编码规范**：脚本文件生成与读写必须强制指定 `-Encoding UTF8`，防止中文字符或特殊符号乱码。
5. **管道与对象优先**：在处理数据解析时，优先使用 PowerShell 的对象管道特性（如 `Select-Object`, `Where-Object`），避免过度依赖传统的文本截取。

## Windows Shell 命令执行细则

在 Windows 执行 shell 命令时优先使用 PowerShell 7：

- 直接使用 `pwsh -NoProfile -Command`，不要预先解析可执行文件位置，也不要写死 WindowsApps 中包含版本号的完整路径。只有 `pwsh` 实际执行失败时才检查环境。
- 如果外层 shell 也是 PowerShell，传给 `-Command` 的脚本优先用单引号包住，避免 `$变量` 被外层提前展开。
- 不要在 PowerShell 字符串里用 Bash 风格的 `\"` 转义双引号；PowerShell 不按 Bash 规则处理它。
- 执行 npm 时直接使用 `npm <command>`，例如 `npm run build`。禁止写成 `& 'C:\Program Files\nodejs\npm.ps1' <command>`：npm 的 PowerShell wrapper 会错误解析 call operator `&`，截坏单引号参数并触发 `Invoke-Expression: The string is missing the terminator`。
- `rg` 正则包含 `|`、`(`、`)`、`\` 等字符时，用单引号包住 pattern，避免 `|` 被解析成 PowerShell 管道。
- 需要在 `-Command` 脚本中使用 `$p/$i/$lines` 等变量时，必须确保这些 `$` 没有被外层 PowerShell 展开：优先使用外层单引号，或用反引号转义 `$`。
- 尽量避免嵌套多层引号；复杂命令拆成更简单的单个命令执行。
- `foreach (...) { ... } | Format-Table` 这类写法在 `-Command` 中容易触发 `An empty pipe element is not allowed`。应先收集结果再管道输出：`$results = foreach (...) { ... }; $results | Format-Table -AutoSize`。
- PowerShell 不支持 Bash heredoc：禁止写 `python - <<'PY' ... PY`。短脚本用 `python -c '...'`，长脚本用 PowerShell here-string 后再传给 Python。
- `Test-NetConnection` 可能明显超过预期等待时间；需要快速端口探测时，优先用 `[System.Net.Sockets.TcpClient]` + `AsyncWaitHandle.WaitOne(timeout)` 控制超时。
- 对多个可选命令做探测时，不要直接依赖 `Get-Command a,b,c -ErrorAction SilentlyContinue` 的退出码；缺少其中一个命令也可能让整体命令返回失败。改为逐个命令检查并输出对象结果。
- 多 URL HTTP 探测不要一次塞进长循环后才输出；每个请求设置 `-TimeoutSec`，必要时拆成多条命令，避免外层工具超时吞掉后续结果。
- 读取带行号的文件时，使用如下模板：

```powershell
pwsh -NoProfile -Command '$p="models\file.py"; $lines=Get-Content -Path $p; for($i=1;$i -le 80;$i++){ "{0,4}: {1}" -f $i,$lines[$i-1] }'
```

- 使用 `rg` 搜索时，使用如下模板：

```powershell
pwsh -NoProfile -Command 'rg -n "simple_text" models configs docs'
```

- 如果正则里有管道或括号：

```powershell
pwsh -NoProfile -Command 'rg -n ''audible|Audible|sound_prob|pred_sound_prob'' models configs docs'
```
## 六、破坏性命令防御

涉及删除、覆盖、迁移、回滚、重置、清库等操作时，先做自检。适用命令包括但不限于：

- `rm`
- `Remove-Item`
- `rmdir`
- `del`
- `Move-Item` 覆盖
- `DROP TABLE`
- `TRUNCATE`
- `DELETE FROM`
- `git reset --hard`
- `git clean`

### 6.1 执行前强制检查

1. 展开路径后再校验。不要直接执行含 `~`、`$VAR`、`%VAR%` 的删除命令。
2. 删除前先列目录内容：`ls -la <target>` 或 `Get-ChildItem <target>`。
3. 看到的内容数量级和预期不一致时，停止。
4. 使用绝对路径，避免相对路径误删当前工作目录。
5. 父目录是 `/`、`~`、用户目录、磁盘根目录、`.ssh`、`.git`、`.claude`、`AppData` 等敏感位置时，停止并重新确认。
6. 命令展开后明显变短，通常表示变量被吃掉，停止。
7. 中文路径参与破坏性操作时，必须先验证 `Test-Path` 或 `[ -e ]`，并使用单一 shell 完成。

### 6.2 禁止跨 shell 套娃

不要构造：

- Bash → PowerShell → cmd → rmdir
- Bash → PowerShell → SSH → bash
- Python `subprocess.run(cmd, shell=True)` 拼字符串

需要 Python 调用命令时，用参数数组：

python
subprocess.run(["rm", path], check=True)


不要用：
python
subprocess.run(f"rm {path}", shell=True)


### 6.3 SSH 远程命令

`ssh host "rm -rf $LOCAL_VAR/foo"` 中 `$LOCAL_VAR` 会在本地展开，不是在远程展开。需要远程展开时，整段远程命令用单引号。

任何让你犹豫的破坏性命令，先停下，不要执行。
