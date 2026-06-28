# ubuntu操作系统初始化

## 写在前面

1. ubuntu操作系统主要有两类：desktop和server。目前由于使用的是阿里的无影云电脑，它默认选择安装的镜像只有ubuntu22.04，是桌面版的linux，所以以下的描述会基于该版本和前提进行展开。

## 预安装软件删繁就简

1. 使用以下命令卸载不必要的软件

```bash
sudo apt purge --autoremove gnome-games libreoffice* thunderbird rhythmbox shotwell totem transmission
sudo snap remove snap-store    #sanp图形商店，可能本来就没有
```

2. 清理包管理器缓存

   Ubuntu 会缓存下载的 `.deb` 安装包，这些可以安全删除

   ```bash
   sudo apt clean          # 清理所有缓存包
   sudo apt autoclean      # 仅清理过时缓存
   ```

3.  删除不再需要的旧内核

   系统升级会保留多个内核，只保留最新两个即可

   ```bash
   sudo apt autoremove --purge   # 自动移除旧内核及无用依赖
   ```

4. 清理临时文件

   ```bash
   sudo rm -rf /tmp/*      # 谨慎操作，可能清理正在使用的临时文件
   ```

5. 查看大文件占用

   找出根目录下占用空间较大的目录

   ```bash
   sudo du -h / --max-depth=1 | sort -hr | head -10
   ```

6. 卸载LXD（如果不用容器）

   ```bash
   sudo snap remove lxd
   ```

7. **结束之后镜像备份**

## 桌面布局

1. 左侧Dock栏自动隐藏

   settings-appliance-dock:勾选auto-hide the dock选项

2. 桌面背景图更换

   settings-background

## bashrc配置

1. 在Ubuntu中设置Tab键自动补全时不区分大小写

   在终端中执行以下命令

   ```bash
   echo "bind 'set completion-ignore-case on'" >> ~/.bashrc
   ```

   这条命令会向 `~/.bashrc` 文件中追加一行配置，修改完后执行执行 `source ~/.bashrc`

## 安装基本的软件工具

首先**更新软件包列表**

```bash
sudo apt update
sudo apt upgrade
```



1. 安装vim-gtk3

   ```bash
   sudo apt install vim-gtk3
   ```

2. 安装git

   ```bash
   sudo apt install git
   git --version
   ```

   > [!TIP]
   >
   > Git 安装完成后，强烈建议马上设置你的用户名和邮箱，因为每次 Git 提交都会用到这些信息。
   >
   > ```bash
   > # 设置你的用户名（可以是昵称或真实姓名）
   > git config --global user.name "你的名字"
   > 
   > # 设置你的邮箱（建议与 GitHub/GitLab 等托管平台一致）
   > git config --global user.email "你的邮箱@example.com"
   > ```
   >
   > 
   >
   > 如果想确认配置是否生效，可以运行：
   >
   > ```bash
   > git config --global --list
   > ```

3. python3和pip默认已安装

   查看版本

   ```bash
   python3 --version
   pip3 --version
   ```

4. 安装并配置claude code

   4.1 安装node.js

   ```bash
   # 1.添加node20源 
   curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash - 
   sudo apt install nodejs -y
   ```

   验证安装

   ```bash
   node -v
   npm -v
   ```

   配置NPM国内镜像

   ```bash
   npm config set registry https://registry.npmmirror.com
   npm config get registry
   ```

   如果输出 `https://registry.npmmirror.com/`，说明镜像源切换成功了

   4.2 安装claude code

   ```bash
   sudo npm install -g @anthropic-ai/claude-code
   #更新npm版本
   sudo npm install -g npm@11.17.0
   ```

   验证安装

   ```bash
   claude --version
   ```

   4.3 配置环境变量以接入deekseek

   将以下配置添加到.bashrc中

   ```bash
   # 将 <你的 DeepSeek API Key> 替换为真实的 API Key
   export ANTHROPIC_BASE_URL="https://api.deepseek.com/anthropic"
   export ANTHROPIC_AUTH_TOKEN="<你的 DeepSeek API Key>"
   export ANTHROPIC_MODEL="deepseek-v4-pro[1m]"
   export ANTHROPIC_DEFAULT_OPUS_MODEL="deepseek-v4-pro[1m]"
   export ANTHROPIC_DEFAULT_SONNET_MODEL="deepseek-v4-pro[1m]"
   export ANTHROPIC_DEFAULT_HAIKU_MODEL="deepseek-v4-flash"
   export CLAUDE_CODE_SUBAGENT_MODEL="deepseek-v4-flash"
   export CLAUDE_CODE_EFFORT_LEVEL="max"
   ```

   执行

   ```bash
   source ~/.bashrc
   ```

   4.4 启动claude

   直接输入claude,如果看到 Claude Code 的欢迎界面和提示符，说明配置已经成功

   4.5 问题修复（可选）

   如果启动claude后，**提示：Auto-update failed: no write permission to npm prefix ，按如下操作：**

   ```bash
   # 创建用户级别的 npm 全局目录
   mkdir -p ~/.npm-global
   
   # 配置 npm 使用该目录
   npm config set prefix '~/.npm-global'
   
   # 将该目录的 bin 子目录添加到 PATH 环境变量
   
   # 对于 bash 用户：
   echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
   source ~/.bashrc
   ```

   完成上述配置后，重新安装 Claude Code 即可。

   ```bash
   npm install -g @anthropic-ai/claude-code
   ```

   

## vimrc配置

逐步更新......

打开.vimrc,没有的话需要创建

```bash
gvim ~/.vimrc
```

.vimrc内容：

```
set mouse=a
set mousemodel=popup
colorscheme desert
set number
```

## 中文输入法支持

1. 直接使用ubuntu系统自带的ibus就可以
   - settings-region & language-点击manage installed languages更新
   - 桌面右上角点击En—选择chinese(pinyin)
   - 按shift切换中英文输入