---

# Git/Gitee新手避坑指南：38个常见问题与解决方案

本文档为 Git/Gitee 新手整理了 38 个常见问题，每个问题均包含**现象描述**、**原因分析**、**解决步骤**及**具体操作案例**。案例中的操作过程以命令和输出形式呈现，便于读者对照自身情况。


| 序号 | 类别 | 问题描述 |
|------|------|----------|
| 1 | 注册与账号 | 注册时收不到验证邮件 |
| 2 | 注册与账号 | 忘记 Gitee 登录密码 |
| 3 | 注册与账号 | 修改用户名后本地仓库无法推送 |
| 4 | 注册与账号 | 生成 SSH 密钥失败 |
| 5 | 注册与账号 | 配置 Git 时权限或路径错误 |
| 6 | 准备工作 | 安装 Git 后右键无“Git Bash Here” |
| 7 | 准备工作 | 邮箱填错导致头像和贡献度不显示 |
| 8 | 代码上传 | SSH 密钥配置失败 |
| 9 | 代码上传 | 首次推送 src refspec 错误 |
| 10 | 代码上传 | 推送被拒（远程有更新） |
| 11 | 代码上传 | HTTPS 推送认证失败 |
| 12 | 代码上传 | 分支被保护无法推送 |
| 13 | 代码上传 | remote origin already exists |
| 14 | 代码上传 | LF 将被 CRLF 替换警告 |
| 15 | 代码上传 | 本地分支名与远程不一致 |
| 16 | 分支与协作 | git pull 后合并冲突 |
| 17 | 分支与协作 | 提交到了错误分支 |
| 18 | 分支与协作 | 撤销已推送的提交 |
| 19 | 分支与协作 | 本地有未提交修改时拉取 |
| 20 | 分支与协作 | git pull --rebase 卡死 |
| 21 | 分支与协作 | git rebase --abort 失败 |
| 22 | Gitee 特有 | 保护分支下如何提交 |
| 23 | Gitee 特有 | Gitee Pages 样式丢失或 404 |
| 24 | Git 基础 | git add 与 commit 混淆 |
| 25 | Git 基础 | .gitignore 不生效 |
| 26 | Git 基础 | 提交后漏文件或信息错误 |
| 27 | Git 基础 | 撤销已推送提交（revert） |
| 28 | 环境细节 | 每次推送都要输入密码 |
| 29 | 环境细节 | 换行符导致文件被标记修改 |
| 30 | 环境细节 | 中文文件名乱码 |
| 31 | 环境细节 | 同步 Fork 仓库 |
| 32 | 环境细节 | 为他人项目贡献代码（PR） |
| 33 | 平台细节 | 创建仓库时选项如何选择 |
| 34 | 平台细节 | 仓库容量超限移除大文件 |
| 35 | 平台细节 | Gitee Pages 绑定自定义域名 |
| 36 | 远程补充 | 缺少 HTTPS helper |
| 37 | 远程补充 | 推送时 Broken pipe |


---

## 第一部分：注册与账号基础问题

### 问题1：注册时收不到验证邮件

**现象**  
用户在 Gitee 官网填写用户名、邮箱、密码后点击注册，但邮箱未收到激活邮件。

**原因**  
- 邮件被误判为垃圾邮件。  
- 使用了延迟较高的企业邮箱。  
- 网络波动导致发送延迟。

**解决步骤**  
1. 检查邮箱的“垃圾邮件”文件夹。  
2. 若未找到，返回登录页点击“重新发送激活邮件”。  
3. 更换为常用邮箱（如 QQ 邮箱、163 邮箱、Gmail）重新注册。

**具体案例**  
操作记录：  
1. 用户使用公司邮箱 `user@company.com` 注册 Gitee，点击“注册”后未收到邮件。  
2. 打开邮箱“垃圾邮件”文件夹，发现激活邮件，点击链接激活成功。  
3. 若垃圾箱仍无邮件，用户返回登录页点击“重新发送激活邮件”，等待 10 分钟后仍未收到。  
4. 改用 QQ 邮箱 `user@qq.com` 重新注册，立即收到邮件并成功激活。

---

### 问题2：忘记了 Gitee 登录密码

**现象**  
用户尝试登录 Gitee 时提示密码错误，无法登录。

**原因**  
长时间未登录或密码记忆模糊。

**解决步骤**  
1. 在登录页点击“忘记密码”。  
2. 输入绑定的邮箱或用户名。  
3. 前往邮箱点击重置链接，设置新密码。

**具体案例**  
操作记录：  
1. 用户输入用户名 `developer` 后提示密码错误，点击“忘记密码”。  
2. 输入注册邮箱 `developer@163.com`，Gitee 发送重置邮件。  
3. 用户登录邮箱，点击邮件中的重置链接，输入新密码 `Dev@2025` 并确认。  
4. 返回登录页，使用新密码登录成功。

---

### 问题3：修改 Gitee 用户名后，本地仓库无法推送

**现象**  
用户在 Gitee 设置中修改了用户名，之后执行 `git push` 时出现认证失败或找不到仓库的错误。

**原因**  
本地仓库的远程 URL 仍包含旧用户名，需要更新。

**解决步骤**  
1. 查看当前远程地址：`git remote -v`  
2. 修改远程 URL：`git remote set-url origin 新地址`

**具体案例**  
操作记录：  
1. 用户将 Gitee 用户名从 `olduser` 改为 `newuser`。  
2. 在本地仓库执行 `git push`，报错：  
   ```
   remote: Invalid username or password.
   fatal: Authentication failed for 'https://gitee.com/olduser/my-project.git/'
   ```  
3. 执行 `git remote -v`，输出：  
   ```
   origin  https://gitee.com/olduser/my-project.git (fetch)
   origin  https://gitee.com/olduser/my-project.git (push)
   ```  
4. 更新远程地址：  
   `git remote set-url origin https://gitee.com/newuser/my-project.git`  
5. 再次推送，输入正确凭据后成功。

---

### 问题4：生成 SSH 密钥时提示“Saving key failed: No such file or directory”

**现象**  
执行 `ssh-keygen -t rsa -C "邮箱"` 后，终端提示：  
```
Saving key "//.ssh/id_rsa" failed: No such file or directory
```

**原因**  
当前用户主目录下不存在 `.ssh` 文件夹，且命令无法自动创建。

**解决步骤**  
1. 手动创建 `.ssh` 目录：`mkdir ~/.ssh`  
2. 重新执行密钥生成命令，一路回车。

**具体案例**  
操作记录：  
1. 用户执行 `ssh-keygen -t rsa -C "user@example.com"`，输出上述错误。  
2. 执行 `mkdir ~/.ssh` 创建目录。  
3. 再次执行 `ssh-keygen -t rsa -C "user@example.com"`，成功生成密钥。

---

### 问题5：配置 Git 时提示“could not lock config file”或“Permission denied”

**现象**  
执行 `git config --global user.name "用户名"` 时报错：  
```
error: could not lock config file D:/xxx/.gitconfig: No such file or directory
```
或  
```
error: could not lock config file D:/Git/.gitconfig: Permission denied
```

**原因**  
- 第一种：`.gitconfig` 文件路径不存在或环境变量配置错误。  
- 第二种：当前用户对 Git 安装目录无写入权限。

**解决步骤**  
1. 对于路径不存在：检查环境变量 `HOME` 或 `USERPROFILE` 是否正确指向用户主目录。  
2. 对于权限问题：以管理员身份运行 Git Bash，或修改 `.gitconfig` 所在目录的权限。

**具体案例**  
操作记录：  
1. 用户执行 `git config --global user.name "developer"`，报错：  
   `error: could not lock config file D:/Git/.gitconfig: Permission denied`  
2. 右键 Git Bash 图标，选择“以管理员身份运行”。  
3. 重新执行配置命令，成功。

---

## 第二部分：准备工作阶段

### 问题6：安装 Git 后，右键找不到“Git Bash Here”

**现象**  
Windows 系统安装 Git 后，在文件夹右键菜单中未出现“Git Bash Here”选项。

**原因**  
安装时未勾选“Windows Explorer integration”选项。

**解决步骤**  
1. 重新运行 Git 安装包。  
2. 选择“Modify”，在组件列表中勾选“Git Bash Here”和“Git GUI Here”。  
3. 完成安装。

**具体案例**  
操作记录：  
1. 用户从官网下载 Git 安装包并默认安装。  
2. 在桌面新建文件夹 `project`，右键菜单无“Git Bash Here”。  
3. 重新运行安装程序，进入“Select Components”界面，展开“Windows Explorer integration”，勾选“Git Bash Here”和“Git GUI Here”。  
4. 完成安装后，右键菜单中正常显示“Git Bash Here”。

---

### 问题7：配置 Git 用户信息时邮箱填错，导致 Gitee 头像和贡献度不显示

**现象**  
代码推送后，Gitee 仓库提交记录中用户头像为默认灰色，且贡献度未增加。

**原因**  
本地 Git 配置的邮箱与 Gitee 账号绑定的邮箱不一致。

**解决步骤**  
1. 检查当前配置：`git config user.email`  
2. 修改为 Gitee 绑定的邮箱：`git config --global user.email "正确的邮箱"`  
3. 修复历史提交（可选）：  
   - 修改最后一次提交：`git commit --amend --reset-author -m "新信息"`  
   - 强制推送（确保分支仅本人使用）：`git push --force`

**具体案例**  
操作记录：  
1. 用户本地配置：  
   ```
   git config --global user.name "developer"
   git config --global user.email "dev@personal.com"
   ```  
2. 推送后 Gitee 提交记录显示未关联用户。  
3. 登录 Gitee，在“设置”->“邮箱管理”中确认注册邮箱为 `dev@company.com`。  
4. 修改本地配置：  
   `git config --global user.email "dev@company.com"`  
5. 修正最后一次提交：  
   `git commit --amend --reset-author -m "修正作者邮箱"`  
6. 强制推送：`git push --force`  
7. 头像和贡献度正常显示。

---

## 第三部分：代码上传核心问题

### 问题8：SSH 密钥配置失败，克隆时报错 Permission denied (publickey)

**现象**  
执行 `git clone git@gitee.com:用户名/仓库名.git` 时返回：  
```
git@gitee.com: Permission denied (publickey).
fatal: Could not read from remote repository.
```

**原因**  
- 未生成 SSH 密钥。  
- 公钥未添加到 Gitee。  
- 私钥权限设置不当。

**解决步骤**  
1. 检查 SSH 密钥：`ls -al ~/.ssh`  
2. 生成新密钥：`ssh-keygen -t rsa -b 4096 -C "邮箱"`  
3. 复制公钥：`cat ~/.ssh/id_rsa.pub`  
4. 登录 Gitee，在“设置”->“SSH 公钥”中添加。  
5. 测试连接：`ssh -T git@gitee.com`

**具体案例**  
操作记录：  
1. 用户执行 `git clone git@gitee.com:user/test.git`，报错。  
2. 检查 SSH 目录：`ls -al ~/.ssh`，发现无 `id_rsa.pub`。  
3. 生成密钥：`ssh-keygen -t rsa -b 4096 -C "user@example.com"`，一路回车。  
4. 复制公钥：`cat ~/.ssh/id_rsa.pub`，复制输出内容。  
5. 登录 Gitee，进入“设置”->“SSH 公钥”，粘贴公钥并保存。  
6. 测试连接：`ssh -T git@gitee.com`，输出 `Hi 用户名! You've successfully authenticated...`  
7. 再次克隆，成功。

---

### 问题9：首次推送代码，报错 src refspec master does not match any

**现象**  
执行 `git push origin master` 报错：  
```
error: src refspec master does not match any
error: failed to push some refs to '...'
```

**原因**  
本地尚未进行任何提交，master 分支不存在。

**解决步骤**  
1. 添加文件并提交：`git add .` → `git commit -m "first commit"`  
2. 再次推送。

**具体案例**  
操作记录：  
1. 用户执行：  
   ```
   git init
   git remote add origin https://gitee.com/user/myapp.git
   ```  
2. 直接推送：`git push origin master`，报错。  
3. 创建初始提交：  
   ```
   echo "# MyApp" > README.md
   git add README.md
   git commit -m "Initial commit"
   ```  
4. 再次推送：`git push origin master`，成功。

---

### 问题10：推送被拒绝，报错 failed to push some refs to，提示远程有本地没有的更新

**现象**  
执行 `git push` 报错：  
```
! [rejected]        master -> master (fetch first)
error: failed to push some refs to '...'
```

**原因**  
远程仓库存在本地没有的提交（如自动生成的 README），需要先合并。

**解决步骤**  
1. 拉取远程代码并允许合并不相关历史：  
   `git pull origin master --allow-unrelated-histories`  
2. 解决冲突（如有），保存合并信息。  
3. 再次推送。

**具体案例**  
操作记录：  
1. 用户在 Gitee 网页创建仓库时勾选了“初始化仓库”，生成了 README.md。  
2. 本地操作：  
   ```
   git init
   echo "笔记内容" > note.txt
   git add note.txt
   git commit -m "add note"
   git remote add origin https://gitee.com/user/my-note.git
   ```  
3. 推送报错。  
4. 执行拉取：  
   `git pull origin master --allow-unrelated-histories`  
   输出：  
   ```
   Merge made by the 'recursive' strategy.
   README.md | 1 +
   1 file changed, 1 insertion(+)
   ```  
5. 再次推送：`git push origin master`，成功。

---

### 问题11：HTTPS 推送时提示 Authentication failed

**现象**  
使用 HTTPS 推送时弹出登录框，输入正确用户名和密码后仍提示认证失败。

**原因**  
Gitee 已不再支持直接使用账号密码进行 HTTPS 推送，需使用私人令牌。

**解决步骤**  
1. 登录 Gitee，进入“设置”->“私人令牌”，生成新令牌（勾选必要权限）。  
2. 复制令牌。  
3. 再次推送，用户名输入 Gitee 用户名，密码输入令牌。

**具体案例**  
操作记录：  
1. 用户执行 `git push origin master`，弹出登录窗口。  
2. 输入用户名 `devuser` 和 Gitee 登录密码，认证失败。  
3. 登录 Gitee，生成私人令牌（如 `f1a2b3c4d5e6...`）。  
4. 再次推送，用户名输入 `devuser`，密码输入令牌，成功。

---

### 问题12：git push 时报错因分支被保护

**现象**  
执行 `git push origin master` 被拒绝，提示分支受保护。

**原因**  
项目管理者将 master 分支设置为保护分支，禁止直接推送，需通过 Pull Request。

**解决步骤**  
1. 本地创建新分支：`git checkout -b feature/new`  
2. 在新分支上开发并提交。  
3. 推送新分支：`git push origin feature/new`  
4. 在 Gitee 网页上发起 Pull Request，请求合并到 master。

**具体案例**  
操作记录：  
1. 用户推送 master 分支，报错：  
   ```
   remote: error: GE007: Your push would publish a private email address.
   remote: error: hook declined to update refs/heads/master
   ```  
2. 创建功能分支并推送：  
   ```
   git checkout -b feature/fix-typo
   # 修改文件
   git add .
   git commit -m "fix typo"
   git push origin feature/fix-typo
   ```  
3. 登录 Gitee，进入仓库，点击“Pull Requests”->“新建 Pull Request”，选择源分支 `feature/fix-typo`，目标分支 `master`，提交。  
4. 等待管理员审核合并。

---

### 问题13：执行 git push 时提示“remote origin already exists”

**现象**  
执行 `git remote add origin <仓库地址>` 报错：  
```
error: remote origin already exists.
```

**原因**  
本地已存在名为 origin 的远程仓库配置。

**解决步骤**  
1. 查看现有远程：`git remote -v`  
2. 删除原有 origin：`git remote rm origin`  
3. 重新添加：`git remote add origin <新地址>`

**具体案例**  
操作记录：  
1. 用户执行 `git remote add origin https://gitee.com/user/project.git`，报错。  
2. 执行 `git remote -v`，显示原有地址。  
3. 删除：`git remote rm origin`  
4. 重新添加：`git remote add origin https://gitee.com/user/project.git`  
5. 验证：`git remote -v` 显示新地址。

---

### 问题14：git add 时提示“warning: LF will be replaced by CRLF”

**现象**  
在 Windows 上执行 `git add` 时，终端输出警告：  
```
warning: LF will be replaced by CRLF in <文件名>
```

**原因**  
Windows 使用 CRLF 换行符，Linux/macOS 使用 LF，Git 默认在提交时进行转换。

**解决步骤**  
全局设置 `core.autocrlf`：  
- Windows：`git config --global core.autocrlf true`  
- 禁用自动转换（不推荐）：`git config --global core.autocrlf false`

**具体案例**  
操作记录：  
1. 用户在 Windows 上执行 `git add .`，输出警告。  
2. 执行 `git config --global core.autocrlf true`  
3. 重新 `git add`，警告消失。

---

### 问题15：本地分支名与远程分支名不一致（main vs master）

**现象**  
执行 `git push` 报错：`error: src refspec master does not match any`

**原因**  
本地默认分支为 main，远程为 master（或反之），名称不一致。

**解决步骤**  
1. 查看当前分支：`git branch`  
2. 重命名分支：  
   ```
   git branch -m main master
   ```  
3. 推送并设置上游：`git push -u origin master`  
4. （可选）修改默认分支名：`git config --global init.defaultBranch master`

**具体案例**  
操作记录：  
1. 用户执行 `git push`，报错。  
2. 执行 `git branch`，显示 `* main`。  
3. 重命名：`git branch -m main master`  
4. 推送：`git push -u origin master`，成功。

---

## 第四部分：分支与协作问题

### 问题16：git pull 后出现合并冲突

**现象**  
执行 `git pull` 后，提示：  
```
Auto-merging <文件名>
CONFLICT (content): Merge conflict in <文件名>
Automatic merge failed; fix conflicts and then commit the result.
```

**原因**  
两人修改了同一文件的同一区域，Git 无法自动合并。

**解决步骤**  
1. 查看冲突文件：`git status`  
2. 手动编辑文件，删除 `<<<<<<<`、`=======`、`>>>>>>>` 标记，保留最终内容。  
3. 标记为已解决：`git add <文件名>`  
4. 提交合并：`git commit -m "resolve conflict"`  
5. 推送。

**具体案例**  
操作记录：  
1. 开发者 A 先修改 `index.html` 并推送。  
2. 开发者 B 在本地也修改了同一文件，推送时被拒绝。  
3. B 执行 `git pull`，出现冲突提示。  
4. 打开 `index.html`，看到：  
   ```
   <<<<<<< HEAD
   <h1>Welcome</h1>
   =======
   <h1>首页</h1>
   >>>>>>> commit-hash
   ```  
5. 沟通后决定保留“首页”，删除标记行，保存文件。  
6. 执行：  
   ```
   git add index.html
   git commit -m "解决标题冲突"
   ```  
7. 推送成功。

---

### 问题17：不小心把代码提交到了错误的分支

**现象**  
本应在功能分支开发，却在 master 分支上做了提交。

**解决步骤（未推送）**  
1. 撤销最后一次提交但保留修改：`git reset HEAD~1 --soft`  
2. 切换到正确分支：`git checkout feature/branch`  
3. 重新提交：`git commit -m "正确信息"`

**具体案例**  
操作记录：  
1. 用户当前在 master 分支，修改代码并提交：  
   ```
   git add .
   git commit -m "add payment feature"
   ```  
2. 发现应在新分支开发，执行：  
   `git reset HEAD~1 --soft`  
3. 创建并切换到新分支：`git checkout -b feature/pay`  
4. 重新提交：  
   ```
   git add .
   git commit -m "feat: 添加支付功能"
   ```  
5. 推送：`git push origin feature/pay`

---

### 问题18：如何撤销已经推送到远程的提交

**现象**  
需要回退到之前某个版本，但提交已推送。

**解决步骤（两种方式）**  

**方法一：reset + force（适合独自使用的分支）**  
1. 找到目标哈希：`git log --oneline`  
2. 本地回退：`git reset --hard <哈希>`  
3. 强制推送：`git push --force`

**方法二：revert（适合多人协作）**  
1. 生成反向提交：`git revert <坏提交哈希>`  
2. 推送：`git push`

**具体案例（reset 方式）**  
操作记录：  
1. 用户推送了坏提交 `abc123`。  
2. 查看历史：`git log --oneline`，找到稳定版本 `def456`。  
3. 执行：  
   ```
   git reset --hard def456
   git push --force
   ```

**具体案例（revert 方式）**  
操作记录：  
1. 用户执行 `git revert abc123`，生成反向提交。  
2. 保存提交信息。  
3. 推送：`git push origin master`

---

### 问题19：本地有未提交的修改，如何拉取远程更新而不丢失本地改动

**现象**  
执行 `git pull` 报错：  
```
error: Your local changes would be overwritten by merge.
```

**解决步骤**  
1. 暂存本地修改：`git stash`  
2. 拉取更新：`git pull`  
3. 恢复修改：`git stash pop`  
4. 若产生冲突，解决后执行 `git stash drop`

**具体案例**  
操作记录：  
1. 用户修改了 `config.py` 未提交，执行 `git pull` 报错。  
2. 执行：  
   ```
   git stash
   git pull
   git stash pop
   ```  
3. 无冲突，恢复成功。

---

### 问题20：执行 git pull --rebase 后进入卡死或中断状态

**现象**  
执行 `git pull --rebase` 后，提示文件被占用或冲突，状态显示为 `(master|REBASE 1/1)`。

**原因**  
- 文件被其他程序占用（如 PPT 被 PowerPoint 打开）。  
- rebase 过程中断。

**解决步骤**  
1. 关闭占用文件的程序。  
2. 中止 rebase：`git rebase --abort`  
3. 改用普通合并：`git pull origin master --allow-unrelated-histories`  
4. 解决冲突后提交并推送。

**具体案例**  
操作记录：  
1. 用户执行 `git pull --rebase origin master`，输出：  
   ```
   Unlink of file 'Git.pptx' failed. Should I try again? (y/n)
   ```  
2. 关闭正在打开的 PowerPoint 文件。  
3. 执行 `git rebase --abort`  
4. 改用：`git pull origin master --allow-unrelated-histories`  
5. 合并成功后推送。

---

### 问题21：执行 git rebase --abort 失败，提示文件会被覆盖

**现象**  
执行 `git rebase --abort` 报错：  
```
error: The following untracked working tree files would be overwritten by reset: <文件名>
```

**原因**  
工作区中存在与 rebase 目标冲突的未跟踪文件。

**解决步骤**  
1. 备份冲突文件。  
2. 删除或移动这些文件。  
3. 重新执行 `git rebase --abort`  
4. 恢复备份（如有需要）。

**具体案例**  
操作记录：  
1. 用户执行 `git rebase --abort`，报错提示 `Git.pptx` 会被覆盖。  
2. 备份 `Git.pptx` 到桌面，然后删除原文件：`rm Git.pptx`  
3. 再次执行 `git rebase --abort`，成功。  
4. 将备份文件复制回原目录。

---

## 第五部分：Gitee 特有功能问题

### 问题22：设置了“保护分支”后如何提交代码

**现象**  
直接推送被拒绝，提示分支受保护。

**解决步骤**  
1. 创建功能分支。  
2. 推送功能分支。  
3. 在 Gitee 发起 Pull Request。

**具体案例**  
操作记录：  
1. 用户推送 master 被拒，提示分支受保护。  
2. 创建分支并推送：  
   ```
   git checkout -b feature/new
   # 开发
   git push origin feature/new
   ```  
3. 在 Gitee 发起 PR，等待审核。

---

### 问题23：Gitee Pages 开启后，网站样式丢失或 404

**现象**  
Pages 网站可访问，但 CSS、图片等资源加载失败（404）。

**原因**  
资源路径使用了绝对路径，未考虑项目名作为子路径。

**解决步骤**  
将 HTML 中的资源引用改为相对路径（去掉开头的斜杠）。

**具体案例**  
操作记录：  
1. 仓库名为 `myblog`，根目录有 `index.html` 和 `css/style.css`。  
2. 原引用：`<link rel="stylesheet" href="/css/style.css">`  
3. 修改为：`<link rel="stylesheet" href="css/style.css">`  
4. 提交推送，Pages 自动重新部署，样式恢复正常。

---

## 第六部分：Git 基础操作混淆问题

### 问题24：git add 和 git commit 的区别，为什么直接 git commit 报错

**现象**  
修改文件后直接执行 `git commit -m "msg"`，提示 `no changes added to commit`。

**原因**  
修改未通过 `git add` 添加到暂存区。

**解决步骤**  
先 `git add`，再 `git commit`，或使用 `git commit -a`（仅对已跟踪文件）。

**具体案例**  
操作记录：  
1. 用户修改 `README.md`，执行 `git commit -m "update"`，报错。  
2. 执行 `git add README.md`  
3. 再次 `git commit -m "update"`，成功。

---

### 问题25：.gitignore 配置了但文件仍被提交

**现象**  
`.gitignore` 中写入了规则，但某些文件或文件夹仍出现在 `git status` 中。

**原因**  
这些文件在添加 `.gitignore` 之前已被 Git 跟踪。

**解决步骤**  
1. 停止跟踪：`git rm -r --cached <文件/文件夹>`  
2. 提交删除记录。

**具体案例**  
操作记录：  
1. 用户不小心提交了 `node_modules`，之后添加 `.gitignore` 并提交。  
2. `git status` 仍显示 `node_modules` 被修改。  
3. 执行：`git rm -r --cached node_modules`  
4. 提交：`git commit -m "untrack node_modules"`  
5. 后续 `node_modules` 不再被跟踪。

---

### 问题26：刚提交完，发现漏了一个文件或提交信息写错了

**现象**  
提交后发现漏文件或信息有误。

**解决步骤**  
- 漏文件：`git add 漏掉的文件` → `git commit --amend`  
- 改信息：`git commit --amend -m "新信息"`

**具体案例（改信息）**  
操作记录：  
1. 用户执行 `git commit -m "add lonin feature"`  
2. 发现拼写错误，执行 `git commit --amend -m "add login feature"`

**具体案例（漏文件）**  
操作记录：  
1. 用户提交了 `main.py`，忘了 `config.ini`。  
2. 执行：  
   ```
   git add config.ini
   git commit --amend --no-edit
   ```

---

### 问题27：如何撤销已经推送到远程的提交（另解）

**现象**  
需要撤销已推送的提交，但不想破坏历史。

**解决步骤**  
使用 `git revert` 生成反向提交。

**具体案例**  
操作记录：  
1. 用户推送了提交 `abc123`，需要撤销。  
2. 执行 `git revert abc123`，生成反向提交。  
3. 推送：`git push origin master`

---

## 第七部分：环境与协作细节问题

### 问题28：为什么每次 push 都要输入用户名密码？怎么记住？

**现象**  
每次推送都提示输入用户名和令牌。

**解决步骤**  
开启凭据缓存：  
```bash
git config --global credential.helper cache
git config --global credential.helper 'cache --timeout=3600'
```

**具体案例**  
操作记录：  
1. 用户执行 `git config --global credential.helper 'cache --timeout=7200'`  
2. 之后 2 小时内推送无需重复输入。

---

### 问题29：Windows 和 Linux 协同工作，换行符导致所有文件被标记为修改

**现象**  
跨平台切换后，`git status` 显示大量文件被修改，但内容未变。

**原因**  
换行符差异（Windows CRLF vs Linux LF）。

**解决步骤**  
- Windows：`git config --global core.autocrlf true`  
- Linux：`git config --global core.autocrlf input`  
- 重置工作区：  
  ```
  git rm --cached -r .
  git reset --hard
  ```

**具体案例**  
操作记录：  
1. 用户从 Windows 推送代码，在 Linux 上 `git clone` 后执行 `git status`，显示大量文件修改。  
2. 在 Linux 上执行 `git config --global core.autocrlf input`  
3. 重置工作区：  
   ```
   git rm --cached -r .
   git reset --hard
   ```  
4. 再次 `git status`，干净。

---

### 问题30：中文文件名或提交信息显示乱码

**现象**  
`git status` 中中文文件名显示为 `\346\265\213\350\257\225.txt`。

**解决步骤**  
设置 `core.quotepath false`：  
```bash
git config --global core.quotepath false
```

**具体案例**  
操作记录：  
1. 用户添加中文文件“测试.txt”，`git status` 显示乱码。  
2. 执行 `git config --global core.quotepath false`  
3. 再次 `git status`，显示正常。

---

### 问题31：如何同步 Fork 的仓库与原仓库

**现象**  
Fork 的仓库落后于原仓库。

**解决步骤**  
1. 添加上游：`git remote add upstream 原仓库地址`  
2. 拉取更新：`git fetch upstream`  
3. 合并：`git checkout master` → `git merge upstream/master`  
4. 推送：`git push origin master`

**具体案例**  
操作记录：  
1. 用户 Fork 了 `https://gitee.com/author/project.git`。  
2. 本地克隆自己的仓库，添加上游：  
   `git remote add upstream https://gitee.com/author/project.git`  
3. 执行：  
   ```
   git fetch upstream
   git checkout master
   git merge upstream/master
   ```  
4. 推送：`git push origin master`

---

### 问题32：如何给别人的项目贡献代码（完整 PR 流程）

**现象**  
不知如何向开源项目提交代码。

**解决步骤**  
1. Fork 原项目  
2. 克隆自己的 Fork  
3. 创建功能分支  
4. 开发并提交  
5. 推送功能分支  
6. 在 Gitee 发起 Pull Request

**具体案例**  
操作记录：  
1. 用户 Fork 项目到自己的账号。  
2. 克隆：`git clone https://gitee.com/user/project.git`  
3. 创建分支：`git checkout -b fix-typo`  
4. 修改并提交：  
   ```
   git add README.md
   git commit -m "fix typo"
   ```  
5. 推送：`git push origin fix-typo`  
6. 在 Gitee 发起 PR。

---

## 第八部分：平台功能易忽略点

### 问题33：创建仓库时，README、.gitignore、许可证怎么选

**现象**  
新手面对选项不知如何选择。

**解决建议**  
- **README**：建议勾选，用于介绍项目。  
- **.gitignore**：根据项目类型选择模板，避免提交无关文件。  
- **许可证**：开源项目必须选择（如 MIT）。

**具体案例**  
操作记录：  
1. 用户创建 Java 项目，在新建仓库页面：  
   - 勾选“使用 README 文件初始化仓库”  
   - .gitignore 选择“Java”  
   - 许可证选择“MIT License”  
2. 创建后仓库自动生成必要文件。

---

### 问题34：Gitee 仓库容量超限，如何移除大文件

**现象**  
推送时提示文件过大或仓库总容量超限。

**解决步骤**  
1. 使用 BFG 或 `git filter-branch` 从历史中删除大文件。  
2. 强制推送。

**具体案例**  
操作记录：  
1. 用户提交了 120MB 的视频文件，推送被拒。  
2. 使用 BFG 工具：  
   ```
   java -jar bfg.jar --delete-files big.mp4 .git
   ```  
3. 清理：  
   ```
   git reflog expire --expire=now --all && git gc --prune=now --aggressive
   ```  
4. 强制推送：`git push --force`

---

### 问题35：如何绑定自定义域名到 Gitee Pages

**现象**  
希望使用自己的域名访问 Pages 网站。

**解决步骤**  
1. 在仓库“服务”->“Gitee Pages”中填写自定义域名。  
2. 在域名 DNS 添加 CNAME 记录指向 `用户名.gitee.io`。

**具体案例**  
操作记录：  
1. 用户 Pages 地址为 `https://user.gitee.io/blog`，拥有域名 `blog.example.com`。  
2. 在仓库 Pages 设置中，自定义域名输入 `blog.example.com`。  
3. 在 DNS 管理后台添加 CNAME 记录：  
   - 主机记录：blog  
   - 记录类型：CNAME  
   - 记录值：user.gitee.io  
4. 解析生效后即可通过 `http://blog.example.com/blog` 访问。

---

## 第九部分：远程仓库补充问题

### 问题36：git push 时报错“fatal: Unable to find remote helper for 'https'”

**现象**  
执行 `git push` 报错：  
```
fatal: Unable to find remote helper for 'https'
```

**原因**  
Git 安装不完整，缺少 HTTPS 协议支持。

**解决步骤**  
重新安装 Git，安装时勾选：  
- “Git from the command line and also from 3rd-party software”  
- “Use the OpenSSL library”

**具体案例**  
操作记录：  
1. 用户推送时报错。  
2. 卸载当前 Git，从官网重新下载安装。  
3. 安装时勾选上述组件。  
4. 安装完成后推送成功。

---

### 问题37：git push 时提示“fatal: the remote end hung up unexpectedly”或“Broken pipe”

**现象**  
推送大文件或较多文件时出现：  
```
Write failed: Broken pipe
fatal: The remote end hung up unexpectedly
```

**原因**  
- 推送数据量过大，超过 Git 缓冲区限制。  
- 网络连接不稳定。

**解决步骤**  
增加缓冲区大小：  
```bash
git config --global http.postBuffer 524288000
```

**具体案例**  
操作记录：  
1. 用户推送包含大文件的项目时出现上述错误。  
2. 执行 `git config --global http.postBuffer 524288000`  
3. 重新推送，成功。

---

