# Windows\-ClaudeCode\+DeepSeek 极简永久部署手册\.md

## 一、前置依赖（必须）

1. 系统：Win10 1809\+ / Win11 64位

2. 安装 **Git for Windows**（全程默认安装）

3. DeepSeek 平台注册充值，获取 API Key（sk\-xxxx）
        

地址：https://platform\.deepseek\.com/
      

---

## 二、PowerShell 一键安装 Claude Code

以 **管理员身份** 打开 PowerShell，依次执行：

1\. 放行脚本策略（仅首次）

```powershell
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
```

输入 **Y** 回车

2\. 官方一键安装

```powershell
irm https://claude.ai/install.ps1 | iex
```

3\. 重启终端，验证是否安装成功

```powershell
claude --version
```

---

## 三、DeepSeek 永久配置（推荐：settings\.json 方式）

路径：`C:\Users\你的用户名\.claude\settings.json`

无文件则新建，粘贴以下内容，**替换自己的 DeepSeek sk 密钥**：

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://api.deepseek.com/anthropic",
    "ANTHROPIC_AUTH_TOKEN": "你的DeepSeek-sk密钥",
    "ANTHROPIC_MODEL": "deepseek-v4-pro",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "deepseek-v4-pro",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "deepseek-v4-pro",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "deepseek-v4-flash",
    "CLAUDE_CODE_SUBAGENT_MODEL": "deepseek-v4-flash",
    "API_TIMEOUT_MS": "600000"
  }
}
```

保存后 **重启终端** 永久生效。

---

## 四、启动方式

```powershell
claude
```

- ✅ **无需 Claude Pro、无需登录Anthropic账号**

- ✅ 全程消耗 DeepSeek 余额

- ✅ 原生支持终端拖拽图片、粘贴截图识图

---

## 五、图片识图用法

1. **拖拽图片**：直接拖入终端，自动生成 `[Image #1]`

2. **粘贴截图**：Win\+Shift\+S 截图 → 终端 Ctrl\+V 粘贴

---

## 六、常用命令

```powershell
claude          # 启动对话终端
claude doctor   # 环境自检
claude update   # 升级客户端
```

---

## 七、切回官方 Claude（备用）

1. 删除 `settings.json` 中所有 env 内容

2. 执行登录：`claude /login`

> （注：部分内容可能由 AI 生成）
