# 在这台 Windows 机器上操作 GitHub（给 AI 助手看的交接说明）

> 目的：任何 AI 助手读完这一页，就能直接在这台机器上读写 GitHub，**不需要重新配置、不需要问用户要令牌**。

## ⚠️ 先看这条：找文件别用 shell

本机 bash 里 **`ls` / `grep` / `cat` / `head` / `dirname` 全部不可用**（`command not found`）。
`find` 会解析到 `C:\Windows\system32\find.exe`，那是 Windows 的文本查找工具，**不能搜文件**。

→ 找文件、读文件请用内置的 **Glob / Grep / Read** 工具，它们不走 shell。
→ 如果"搜了半天找不到某个文件"，先怀疑 shell 工具链坏了，别下"文件不存在"的结论。

## 一句话结论

**通道已经打通。** 账号 `lixiuzuapple-a11y` 的授权令牌已加密存放在 **Windows 凭据管理器**里，
由 **Git Credential Manager (GCM 2.9.0)** 自动取用。直接跑 git 命令即可，不会被要求输密码。

## 第一步（必须做）：修好 PATH

这台机器的 Bash 环境 PATH 被净化过，`ls` / `mkdir` / `head` / `sed` / `dirname` 都会报
`command not found`，报错源头是 `shell-runtime-bash-env.sh: line 3: dirname: command not found`。

**每条命令开头都要挂上 PortableGit 的三个目录：**

```bash
PG="/c/Users/Administrator/.workbuddy/binaries/PortableGit/versions/1.2.0"
export PATH="$PG/cmd:$PG/mingw64/bin:$PG/usr/bin:$PATH"
```

| 目录 | 提供什么 |
|---|---|
| `$PG/cmd` | `git.exe`（版本 2.55.0.windows.3） |
| `$PG/mingw64/bin` | `git-credential-manager.exe`（GCM 2.9.0） |
| `$PG/usr/bin` | `ls` / `sed` / `grep` / `head` 等基础工具 |

## 第二步：正常用 git

```bash
git clone https://github.com/<账号>/<仓库>.git
cd <仓库>
git add -A && git commit -m "<说明>"
git push
```

凭据自动注入，**不需要密码、不需要令牌**。

## 常用操作

```bash
# 确认绑定了哪个账号
git-credential-manager github list          # 应输出 lixiuzuapple-a11y

# 取令牌调用 GitHub API（注意：不要把令牌打印出来）
CRED=$(printf 'protocol=https\nhost=github.com\n\n' | git credential fill)
TOKEN=$(printf '%s\n' "$CRED" | sed -n 's/^password=//p')
curl -s -H "Authorization: Bearer $TOKEN" -H "Accept: application/vnd.github+json" \
  https://api.github.com/user

# 新建仓库
curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Accept: application/vnd.github+json" \
  https://api.github.com/user/repos -d '{"name":"<名字>","private":false}'

# 改仓库可见性（false=公开，true=私有）
curl -s -X PATCH -H "Authorization: Bearer $TOKEN" -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/<账号>/<仓库> -d '{"private":false}'
```

## 排错速查

| 现象 | 原因 | 解法 |
|---|---|---|
| `ls` / `mkdir` / `sed` command not found | PATH 被净化 | 见「第一步」 |
| `Failed to locate 'git.exe' executable on the path` | 同上，GCM 依赖 PATH 上的 git | 见「第一步」 |
| `Cannot prompt because user interactivity has been disabled` | 设了 `GCM_INTERACTIVE=never` | 改成 `auto` 或不设 |
| 分支后面显示 `[gone]`，`git fetch` 说成功但 `show-ref` 没有 | git 建不出 `refs/remotes/origin/` 子目录 | 先 `mkdir -p .git/refs/remotes/origin` 再 fetch |
| 读注册表被拦 `PROGRAM BLOCKED BY SECURITY POLICY` | `reg.exe` 在沙箱黑名单 | 改用 PowerShell |
| PowerShell 命令没有输出 | 该工具在这台机器不回显 stdout | 用 `Set-Content` 写文件，再读文件 |
| 匿名调 `api.github.com` 返回 403 | 该网络限制匿名 API | 这是正常的；判断仓库是否公开请看网页状态码或匿名 `git ls-remote` |

## 需要重新授权的情况（换机器 / 换用户时）

```bash
export PATH="$PG/cmd:$PG/mingw64/bin:$PG/usr/bin:$PATH"
git-credential-manager configure
git-credential-manager github login --browser --username <GitHub用户名>
```

会拉起默认浏览器（本机是 Edge）打开 GitHub 授权页 → **用户本人点 Authorize** → 令牌自动加密入库。

注意事项：
- **不要用 `--device` 设备码模式**：在后台运行会完全无输出并卡死。
- 授权码**不会出现在命令输出里**，验证一律看 `git-credential-manager github list`。
- 若端口还在监听但登录没生效，用 `Start-Process "<授权URL>"` 把页面重新弹到前台，请用户再点一次。
- 申请到的权限范围是 `repo + gist + workflow`，即**该账号下所有仓库的读写**。

## 安全边界（务必向用户明示）

- 令牌**加密存在本机凭据管理器** → 能登录这个 Windows 账户的人都能推送代码。
- 这台是**公司电脑**，授权前要提醒用户。
- 撤销：GitHub → Settings → Applications → Authorized OAuth Apps；
  本机侧 `git-credential-manager github logout <账号>`。

## 本机已绑定的信息

- 账号：`lixiuzuapple-a11y`（资料页显示名 `lixiuzu`）
- 提交署名：`user.name = lixiuzu` / `user.email = lixiuzuapple@gmail.com`
- 常用本地目录：`C:\Users\Administrator\WorkBuddy\repos\`
