# windows git

## git bash中文文件名乱码

1. 修改git全局配置，打开git bash，依次执行以后命令：

```bash
#核心：关闭中文文件名八进制转义（解决 \344\270\255 乱码）
git config --global core.quotepath false
# 提交、日志编码统一 UTF-8
git config --global i18n.logoutputencoding utf-8
git config --global i18n.commitencoding utf-8
```

2. 执行完关闭git bash重新打开，再输入git status

## 换行符冲突（CRLF自动处理）

执行以下命令：

```bash
git config --global core.autocrlf true
git config --global core.safecrlf warn
```

1. **两种换行符**：

- LF（\n）：Linux /macOS/ Git 仓库标准换行

- CRLF（\r\n）：Windows 记事本原生换行

2. **autocrlf = true**

- 检出（checkout）到本地windows:仓库里的LF——》CRLF

- 提交（commit）上传仓库：本地CRLF——》LF

  > [!NOTE]
  >
  > 解决跨系统协作换行符混乱，比如：
  >
  > 1. windows用户编辑保存产生CRLF，但linux服务器、Mac打开看到大量^M多余符号
  > 2. git diff整行全部判定改动（明明内容没变，只是换行不一样）
  > 3. 防止提交一堆（仅仅换行符不同）的无效变更，造成代码对比一片红

3. core.safecrlf warn作用：

   检测换行转换会不会产生冲突：

   - warn: 发现异常换行转换给出警告，但允许提交
   - true: 直接禁止提交，强制你处理换行
   - false: 静默转换，不提醒

## git bash终端界面编码设置

1. 打开git bash，右键-options
2. Text标签
   - Locale: zh_CN
   - Character set : UTF-8