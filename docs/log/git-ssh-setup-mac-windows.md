# Git + GitHub SSH 配置备忘（macOS / Windows）

长期方案：**同一 GitHub 账号 + 每台电脑各一把 SSH 公钥 + remote 使用 SSH 地址**。  
避免 HTTPS 凭据过期、钥匙串里旧 Token 导致 `403` / `Permission denied (publickey)`。

仓库示例：`git@github.com:sylar-qiu/qianniu-plugin.git`

---

## 一、通用概念

| 项目 | 说明 |
|------|------|
| 远程地址 | `git@github.com:用户名/仓库名.git`（SSH），不是 `https://github.com/...` |
| 每台电脑 | **各自** `ssh-keygen` 生成一对 key，在 GitHub **各 Add 一把**（可并存，不要删另一台） |
| 私钥 | 仅在本机 `~/.ssh/`，**永不**提交 Git、不发聊天 |
| 公钥 | `.pub` 文件内容贴到 GitHub → Settings → SSH and GPG keys |
| 验证 | `ssh -T git@github.com` → `Hi 用户名! You've successfully authenticated...` |

GitHub 官方 ED25519 主机指纹（首次连接提示时核对）：

```text
SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU
```

首次连接输入：`yes`

---

## 二、macOS

### 1. 生成密钥（已存在则不要覆盖）

```bash
ssh-keygen -t ed25519 -C "你的邮箱" -f ~/.ssh/id_ed25519_github
```

提示 `already exists` → 输入 **`n`**，不要 overwrite。

### 2. 查看公钥并加到 GitHub

```bash
cat ~/.ssh/id_ed25519_github.pub
```

复制整行 → https://github.com/settings/keys → **New SSH key**

### 3. SSH 配置 `~/.ssh/config`

```sshconfig
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_github
  AddKeysToAgent yes
  UseKeychain yes
```

### 4. 加入 agent（macOS）

```bash
eval "$(ssh-agent -s)"
ssh-add --apple-use-keychain ~/.ssh/id_ed25519_github
```

### 5. 测试

```bash
ssh -T git@github.com
```

### 6. 仓库改用 SSH 并推送

```bash
cd /path/to/qianniu-plugin
git remote set-url origin git@github.com:sylar-qiu/qianniu-plugin.git
git remote -v
git push
```

### macOS 常见问题

| 现象 | 处理 |
|------|------|
| `Permission denied (publickey)` | 公钥未加到 GitHub，或 `config` 里 `IdentityFile` 路径错 |
| `403` 且用 HTTPS | 改用 SSH remote，或更新 PAT；长期推荐 SSH |
| 误执行 overwrite key | 若旧公钥已在 GitHub，需把**新** `.pub` 再 Add 一次 |

---

## 三、Windows（PowerShell）

### 1. 生成密钥

```powershell
ssh-keygen -t ed25519 -C "你的邮箱" -f $env:USERPROFILE\.ssh\id_ed25519_github
```

已存在则 **`n`** 不覆盖。

### 2. 公钥加到 GitHub（与 Mac **并存**）

```powershell
Get-Content $env:USERPROFILE\.ssh\id_ed25519_github.pub
```

→ https://github.com/settings/keys → **New SSH key**（Title 如 `Windows-PC`）

### 3. SSH 配置 `%USERPROFILE%\.ssh\config`

```sshconfig
Host github.com
  HostName github.com
  User git
  IdentityFile C:/Users/你的用户名/.ssh/id_ed25519_github
  IdentitiesOnly yes
```

路径用 **正斜杠**，`你的用户名` 换成实际目录（如 `EDY`）。

### 4. 启动 ssh-agent 并加载私钥（Windows 常漏）

```powershell
Get-Service ssh-agent | Select-Object Status, StartType
Set-Service ssh-agent -StartupType Manual
Start-Service ssh-agent
ssh-add $env:USERPROFILE\.ssh\id_ed25519_github
ssh-add -l
```

`ssh-add -l` 应能看到 ED25519，而不是 `The agent has no identities`。

### 5. 测试

```powershell
ssh -T git@github.com
```

### 6. 关键：让 Git 使用系统 OpenSSH

现象：`ssh -T` 成功，但 `git pull` 仍 `Permission denied (publickey)`。  
原因：PowerShell 的 `ssh` 与 **Git 自带的 ssh** 不是同一个，agent 里的 key Git 用不上。

**长久修复（推荐，执行一次）：**

```powershell
git config --global core.sshCommand "C:/Windows/System32/OpenSSH/ssh.exe -i C:/Users/你的用户名/.ssh/id_ed25519_github -o IdentitiesOnly=yes"
```

仅当前仓库（可选）：

```powershell
cd C:\path\to\qianniu-plugin
git config core.sshCommand "C:/Windows/System32/OpenSSH/ssh.exe -i C:/Users/你的用户名/.ssh/id_ed25519_github -o IdentitiesOnly=yes"
```

### 7. 拉取 / 推送

```powershell
cd C:\path\to\qianniu-plugin
git remote set-url origin git@github.com:sylar-qiu/qianniu-plugin.git
git remote -v
git pull
git push
```

### Windows 常见问题

| 现象 | 处理 |
|------|------|
| `ssh -T` 失败 | 先做 ssh-agent + `ssh-add`，确认 GitHub 已 Add 公钥 |
| `ssh -T` 成功，`git pull` 失败 | 配置 `core.sshCommand`（见上） |
| `git pull` 很慢 | 正常，首次拉含大文件；勿中断；以后增量会快 |
| PowerShell `curl` 报错 | `curl` 是别名，用 `Invoke-RestMethod` 或 `curl.exe` |

---

## 四、两台电脑协作流程

```text
Mac 开发 → git add / commit → git push  →  GitHub  →  git pull  →  Windows（跑 PG / qn-server）
```

1. 两台机 **同一 GitHub 账号**、仓库 **Write** 权限。  
2. **各一把 SSH key**，GitHub 上两把都保留。  
3. `origin` 统一为 `git@github.com:sylar-qiu/qianniu-plugin.git`。  
4. 勿提交 `.env`、API Key、数据库密码。

---

## 五、HTTPS + Token（备用，不推荐长期）

仅当 SSH 暂时修不好时：

```powershell
git remote set-url origin https://github.com/sylar-qiu/qianniu-plugin.git
git pull
```

- 用户名：GitHub 用户名  
- 密码：**Personal Access Token**（Classic 勾选 `repo`），不是登录密码  

Token 会过期，易与钥匙串冲突；修好 SSH 后改回 SSH remote。

---

## 六、检查清单

**Mac**

- [ ] `~/.ssh/id_ed25519_github` 存在  
- [ ] 公钥已在 GitHub  
- [ ] `~/.ssh/config` 指向该私钥  
- [ ] `ssh -T git@github.com` 成功  
- [ ] `git remote -v` 为 `git@github.com:...`  
- [ ] `git push` 成功  

**Windows**

- [ ] 同上 + `ssh-agent` 已启动、`ssh-add -l` 有 key  
- [ ] `git config --global core.sshCommand` 已指向系统 OpenSSH  
- [ ] `git pull` 成功  

---

## 七、与本项目的关系

- 插件与后端在同一仓库：`extension/` + `qn-server/`。  
- Mac push 后，Windows 在同一目录 `git pull` 即可同步 `qn-server`。  
- 后端密钥在 `qn-server/.env`，与 Git SSH **无关**，也不要提交 Git。

---

*文档版本：2026-05 · 适用于 qianniu-plugin 仓库协作*
