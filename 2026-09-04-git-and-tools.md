# 学习笔记 2026-09-04
## 今日内容：
1. Claude code+GLM配置；
2. git基础；

## 学到的知识
1. 快捷键打开vscode命令面板：win+shift+p；
2. 打开命令面板，选择打开用户设置，即可设置claude code的环境变量，设置成智谱；
3. 如何在github上面创建仓库：点击create repository，输入名称和description，还可以设置可见；
4. git相关操作：
```bash
git --version #显示git版本；
git config --global user.name "your github name" #配置身份-github用户名；
git config --global user.email "your email" #配置身份-电子邮箱，这里可以在github设置，不展示真实有限，复制他生成的虚拟邮箱即可；
git config --global --list #验证配置是否成功，若显示name和email即为成功；
git clone https://github.com/用户名/仓库名.git #克隆项目；
cd D:\  #进入D盘，cd 文件夹，就是进入哪个文件夹；
git add . #把改动的文件放入暂存区；
git commit -m "..." #把暂存的版本打包成一个“版本快照”，引号里面为描述内容；
git push #把本地版本上传github
git status #告诉你当前仓库里每个文件处于什么状态;
git status -s        # 简短模式，每个文件一行：
                     # ?? 未跟踪 / M 已修改未暂存 / M(绿) 已暂存
git restore --staged 文件名   # 后悔了，把文件从暂存区拿出来
git checkout -- 文件名        # 更狠：丢弃本地未暂存的修改（慎用）
git log                      # 完整模式：显示作者、邮箱、日期、完整message
git log --oneline --graph    # 带分支图（多分支时能看清分叉合并）
git log --oneline -5         # 只看最近5条
git log -p 文件名            # 显示具体改了哪些行（最详细）
git show a1b2c3d             # 查看某一次提交的完整改动
git config --global pager.log false #一劳永逸关闭 log 的分页器
git branch                    # 查看所有本地分支（当前分支前有 * 号）
git branch exp                # 创建一个叫 exp 的新分支（但还在原地）
git switch exp                # 切换到 exp 分支（新写法，推荐）
git switch -c exp             # 创建 + 切换一步到位
git switch main               # 切回主线
git branch -d exp             # 删除已合并的分支
```
5. 克隆完项目，用vscode打开相应文件夹，创建新一个新文件，写笔记；
6. 提交三连：在 VSCode 里按 Ctrl + ` 打开内置终端（当前目录就是仓库），依次执行：
```bash
git add . 
git commit -m "..." 
git push 
```
7. Git 三大区域：工作区 → 暂存区（add）→ 仓库（commit）→ 远程（push）；


## 今日实践操作
1. 完成vscode中安装claude code插件，并接入GLM；
2. 学习git相关操作，并学会笔记上传github仓库；


