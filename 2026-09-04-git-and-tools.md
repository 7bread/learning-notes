#学习笔记 2026-09-04
##今日内容：
Claude code+GLM配置；
git基础；

##学到的知识
1.快捷键打开vscode命令面板：win+shift+p；
2.打开命令面板，选择打开用户设置，即可设置claude code的环境变量，设置成智谱；
3.如何在github上面创建仓库：点击create repository，输入名称和description，还可以设置可见；
4.git相关操作：
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
```
5.克隆完项目，用vscode打开相应文件夹，创建新一个新文件，写笔记；
6.提交三连：在 VSCode 里按 Ctrl + ` 打开内置终端（当前目录就是仓库），依次执行：
```bash
git add . 
git commit -m "..." 
git push 
```
7.Git 三大区域：工作区 → 暂存区（add）→ 仓库（commit）→ 远程（push）；


##今日实践操作
1.完成vscode中安装claude code插件，并接入GLM；
2.学习git相关操作，并学会笔记上传github仓库；


