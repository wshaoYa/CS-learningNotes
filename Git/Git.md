<img src="https://s2.loli.net/2023/11/04/XWrxCNl6FUc9isu.png" alt="Git-Cheet-Sheet-ByGeekHour"  />

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

# 基本命令

**tip：**一些命令过于复杂时，可用alias起别名方便调用

- `git init`：创建仓库
- `git clone`：克隆远程的一个现有的仓库至本地
- `git status`：查看仓库的状态
- `git add`：添加到暂存区
- `git commit -m "xxxx"`：提交到本地仓库（只会提交暂存区里的文件）
  - `-a`表示commit前自动进行add，`-am`表示 `-a -m`的缩写合并，遂懒省事的写法：`git commit -am "xxx"`
- `git log`：查看仓库提交历史记录
  - 可以使用 `--oneline` 参数来查看简洁的提交记录
- `git ls-files`：查看暂存区内的文件
- `git push <remote> <branch>`: 从**本地仓库**推送更新内容到**远程仓库**
  - 默认remote和branch为orgin和main
- `git pull <远程仓库名> <远程分支名>:<本地分支名>`: 从**远程仓库**拉取更新内容到**本地仓库**
  - 两分支名相同则可省略冒号后面“本地分支名”部分
  - 默认远程仓库名和分支名为orgin和main
  - <img src="https://s2.loli.net/2023/10/31/pkiJzly6dPMuwoQ.png" alt="image-20231031103759649" style="zoom:50%;" />
  - pull时忽视不相关的历史：` --allow-unrelated-histories `
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

<img src="https://s2.loli.net/2023/11/02/mZEraufy4O8TAJl.png" alt="image-20231102102339036" style="zoom:50%;" />

### 合并分支（merge、rebase）

- `git merge branch-name`：合并分支
- `git merge --abort`：中止合并
- `git rebase branch-name`：合并分支
  - 合并后分支没有变动，在main分支上rebase dev后仍处在main分支
  - 当前分支为curDev，目标合并到targetDev中，即可在curDev分支上执行`git rebase targetDev`，git会找到两分支的最近的公共子节点，以此为界限点，把curDev上新的内容接到targetDev中最新提交后面（可抽象的理解为把targetDev中的改动commit直接挤进curDev中，挤的位置为两分支的最近的公共子节点处），即下图中的左侧。
  - 反之，在targetDev上执行`git rebase curDev`，则会把targetDev公共子节点之后的提交接到curDev的最新提交后面，即下图中的右侧。
  - <img src="https://s2.loli.net/2023/11/02/NX7GuC5kaFUno8T.png" alt="image-20231102112401998" style="zoom: 50%;" />

#### merge优缺点

**优点**

- 不会破坏原分支的提交历史，方便回溯和查看

**缺点**

- 会产生额外的提交节点，分支图比较复杂

**适用场景**

简单的把两分支合并，不关心会使之前的分支图变的略微复杂（总体分支图逐渐庞大可能？）

#### rebase优缺点

**优点**

- 不会新增额外的提交记录，形成线性历史，比较直观和干净

**缺点**

- 会改变提交历史，改变了当前分支branch out的节点，**应避免在共享分支使用**

**适用场景**

只有自己在此分支开发，且希望提交历史为线性更加清晰明了

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

# 合并冲突解决

**产生原因：**

- 两个分支未修改同一个文件的同一处位置: Git 自动合并
- 两个分支修改了同一个文件的同一处位置: 产生冲突

**解决办法：**

- 找到冲突部分，手工修改冲突文件，合并冲突内容
- `git add file` 添加暂存区
- `git commit -m "message"`  提交修改

**中止合并：**

`git merge --abort`：当不想继续执行合并操作时可以使用此命令来中止合并过程

# 分支工作流模型（GitFlow）

基于Git的一种工作分支间的使用规范，⽤于在Git上管理软件开发项⽬。

![image-20231104181308539](https://s2.loli.net/2023/11/04/XrCMTVt6R41NSgf.png)

## **核心分支**

分支长期存在

- **主分⽀（master/main）**：
  - 代表了项⽬的稳定版本，每个提交到主分⽀的代码都应该是经过测试和审核的。
  - 生命周期：长期存在
  - tag版本标识：每个节点都要有新的版本tag标识：例如Tag 1.0.1 （主版本.次版本.修订版本），方便追踪和回溯
    - 主版本：主要的功能变化或重大更新。
    - 次版本：一些新的功能、改进和更新，通常不会影响现有功能。
    - 修订版本：一些小的bug修复，安全漏洞补丁等，通常不会更改现有功能和接口。
- **开发分⽀（develop)**：
  - ⽤于⽇常开发。所有的功能分⽀、发布分⽀和修补分⽀都应该从开发分⽀派⽣出来。
  - 生命周期：长期存在
  - 一般合并至main分支时会更新修订版本

## **辅助分支**

分支作用完成后应及时删除

- **功能分⽀（feature）**：
  - ⽤于开发单独的功能或者特性。每个功能分⽀都应该从开发分⽀派⽣，并在开发完成后合并回开发分⽀。
- **发布分⽀（release)**：
  - ⽤于准备项⽬发布。发布分⽀应该从开发分⽀派⽣，并在准备好发布版本后合并回主分⽀和开发分⽀（cherry-pick）。
- **热修复分⽀（hotfix）**：
  - ⽤于修复主分⽀上的紧急问题。热修复分⽀应该从主分⽀派⽣，并在修复完成后，合并回主分⽀和开发分⽀。

## 分支命名

推荐使用带有意义的描述性名称来命名分支

**示例**

- 版本发布分支/Tag : v1.0.0
- 功能分支 : feature-login-page
- 修复分支 : hotfix-#issueid-desc

## 分支管理

定期合并已经成功验证的分支，及时删除已经合并的分支

保持合适的分支数量

为分支设置合适的管理权限

# 子模块

## 添加子模块

```bash
git submodule add <git链接> <路径>
```

## 更新子模块

<img src="https://s2.loli.net/2024/10/02/76f23lnBgGeHwkD.png" alt="image-20241002135728700" style="zoom: 80%;" />



# IDE文件颜色含义

Jetbrain IDE中安装Git以后，代码文件出现不同颜色分别表示的含义：

- 绿色，已经加入控制暂未提交
- 红色，未加入版本控制
- 蓝色，加入，已提交，有改动
- 白色，加入，已提交，无改动
- 灰色：版本控制已忽略文件