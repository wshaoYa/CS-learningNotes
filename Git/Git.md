# 初始化配置

# 新建仓库

# 工作区域/文件状态

![image-20231030162113621](https://s2.loli.net/2023/10/30/y2lDuf3CjBQASWG.png)

![image-20231030162226732](https://s2.loli.net/2023/10/30/M8vjQEposGIP4hc.png)

![image-20231030153341929](https://s2.loli.net/2023/10/30/OsmPL9Wxn3ubVY4.png)

**纠正：**

- 从“已暂存”—>"已修改"，应该是`git restore --staged `；
- 从“已修改”—>“未修改”，应该是`git restore`

# 常用命令

- `git init`：创建仓库

- `git status`：查看仓库的状态
- `git add`：添加到暂存区
- `git commit`：提交到本地仓库（只会提交暂存区里的文件）
- `git log`：查看仓库提交历史记录
  - 可以使用 `--oneline` 参数来查看简洁的提交记录
- `git ls-files`：查看暂存区内的文件

## git reset

![image-20231030155651253](https://s2.loli.net/2023/10/30/5ZnvGtd4R6yQb1p.png)

## git diff

![image-20231030161742926](https://s2.loli.net/2023/10/30/L3HI85v4D2ZKS91.png)

![image-20231030161530817](https://s2.loli.net/2023/10/30/prid9gTxfBZU8Vy.png)

## git rm

(删除后不要忘记提交)

- `rm file; git add file`：先从工作区删除文件,然后再暂存删除内容
- `git rm <file>`：把文件从工作区和暂存区同时删除
- `git rm --cached <file>`:把文件从暂存区删除，但保留在当前工作区中
- `git rm -r *`:递归删除某个目录下的所有子目录和文件

