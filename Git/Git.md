# 初始化配置

配置用户名、邮箱等

- `git config --global user.name xxx` 

- `git config --global user.email xx@qq.com`

# 新建仓库

两种方式

- `git init`：本地创建一个仓库
  - `git remote add <远程仓库别名> <远程仓库地址>`： 本地仓库与远程仓库建立连接
    - 一般远程仓库别名都会取origin
  - `git push -u <远程仓库名> <分支名>`
    - 默认为仓库名和分支名为orgin和main
- `git clone`：克隆远程的一个现有的仓库至本地

# 工作区域

<img src="https://s2.loli.net/2023/10/30/y2lDuf3CjBQASWG.png" alt="image-20231030162113621" style="zoom:50%;" />

<img src="https://s2.loli.net/2023/10/30/M8vjQEposGIP4hc.png" alt="image-20231030162226732" style="zoom:50%;" />

# 文件状态

<img src="https://s2.loli.net/2023/10/30/OsmPL9Wxn3ubVY4.png" alt="image-20231030153341929" style="zoom: 50%;" />

**纠正：**

- 从“已暂存”—>"已修改"，应该是`git restore --staged `；
- 从“已修改”—>“未修改”，应该是`git restore`

<img src="https://s2.loli.net/2023/10/31/6kf27VWvM4PtX9x.png" alt="image-20231031150930824" style="zoom:50%;" />

# 常用命令

- `git init`：创建仓库
- `git clone`：克隆远程的一个现有的仓库至本地
- `git status`：查看仓库的状态
- `git add`：添加到暂存区
- `git commit`：提交到本地仓库（只会提交暂存区里的文件）
- `git log`：查看仓库提交历史记录
  - 可以使用 `--oneline` 参数来查看简洁的提交记录
- `git ls-files`：查看暂存区内的文件
- `git push <remote> <branch>`: 从**本地仓库**推送更新内容到**远程仓库**
  - 默认remote和branch为orgin和main
- `git pull <远程仓库名> <远程分支名>:<本地分支名>`: 从**远程仓库**拉取更新内容到**本地仓库**
  - 两分支名相同则可省略冒号后面“本地分支名”部分
  - 默认远程仓库名和分支名为orgin和main
  - <img src="https://s2.loli.net/2023/10/31/pkiJzly6dPMuwoQ.png" alt="image-20231031103759649" style="zoom:50%;" />
- `git remote -V`: 查看远程仓库
- `git remote add <远程仓库别名> <远程仓库地址>`： 本地仓库与远程仓库建立连接

## 回滚（git reset）

<img src="https://s2.loli.net/2023/10/30/5ZnvGtd4R6yQb1p.png" alt="image-20231030155651253" style="zoom:50%;" />

## 差异查看（git diff）

<img src="https://s2.loli.net/2023/10/30/L3HI85v4D2ZKS91.png" alt="image-20231030161742926" style="zoom:50%;" />

<img src="https://s2.loli.net/2023/10/30/prid9gTxfBZU8Vy.png" alt="image-20231030161530817" style="zoom:50%;" />

## 删除（git rm）

(删除后不要忘记提交)

- `rm file; git add file`：先从工作区删除文件,然后再暂存删除内容
- `git rm <file>`：把文件从工作区和暂存区同时删除
- `git rm --cached <file>`:把文件从暂存区删除，但保留在当前工作区中
- `git rm -r *`:递归删除某个目录下的所有子目录和文件

## 分支（git branch）

- `git branch`：查看分支列表
  - 前带*号的表示当前所处分支
- `git branch branch-name`：创建分支
- `git checkout branch-name`：切换分支（不推荐）
  - checkout同时有恢复restore的一些功能，当分支名和某需要restore的文件重名时，使用此命令会有歧义
- `git switch branch-name`：切换分支（推荐）
- `git merge branch-name`：合并分支
- `git branch -d branch-name`：删除分支（已合并）
- `git branch -D branch-name`：强制删除分支（未合并）

- `git log --graph --oneline --decorate --all`：控制台界面查看分支图



# .gitignore

**注意**：只能忽略暂未存到本地仓库（版本库）的文件，如本地仓库中已有，则即使.gitignore中声明了也没用

**tip**：github上有哪种针对各个编程语言/项目等较为通用的gitignore文件，可下载直接拿来用

## 应该忽略哪些文件

<img src="https://s2.loli.net/2023/10/30/jdKao8Y6Vgk1uXN.png" alt="image-20231030171444092" style="zoom:50%;" />

## 匹配规则

<img src="https://s2.loli.net/2023/10/30/NBxkpoi3Qrfe9tI.png" alt="image-20231030171737817" style="zoom:50%;" />

<img src="https://s2.loli.net/2023/10/30/SZ2DQcxoMt7ln69.png" alt="image-20231030171848292" style="zoom:50%;" />

## 实例

<img src="https://s2.loli.net/2023/10/30/WMvJk1PBiDwr6Sg.png" alt="image-20231030172032740" style="zoom: 50%;" />

# SSH配置

在本地用户目录下的.ssh目录下执行`ssh-keygen -t rsa -b 4096`，生成公钥和私钥文件

- 私钥文件: `id_rsa `
  - 放在本地保存好就好
- 公钥文件: `id_rsa.pub`
  - 使用此文件到github中进行setting设置

配置好后即可直接使用ssh进行仓库的下载等操作

**注意**：默认生成的文件名称为id_rsa（会覆盖之前已有的id_rsa，若有的话），可在生成中途设置自定义名称，但最后记得配置下.ssh目录下的config，例如：

<img src="https://s2.loli.net/2023/10/31/qOLvPHZoNdtng1J.png" alt="image-20231031103458357" style="zoom:50%;" />

## 公钥/私钥

- 每个用户都有一对私钥和公钥。
  - 私钥：用来进行解密和签名，是给自己用的。
  - 公钥：由本人公开，用于加密和验证签名，是给别人用的。
  - 当该用户发送文件时，用私钥签名，别人用他给的公钥解密，可以保证该信息是由他发送的。即数字签名。

- 当该用户接受文件时，别人用他的公钥加密，他用私钥解密，可以保证该信息只能由他看到。即安全传输。

# 代码托管平台

- github
  - 全球最大公有代码托管平台（同性交友平台）
- gitee（码云）
  - 国内平台，对国内用户更友好
- gitlab（极狐）：
  - 私有化部署，**区别于**github和gitee这些**公有的**代码托管平台，其可搭建自己的Gitlab服务器存储项目，安全性高，企业场景使用较多