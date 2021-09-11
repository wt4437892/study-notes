# Git笔记

## Git  配置

- `git config --list` 检查所有配置信息
- `git config <key>` 检查某一项配置

## 获取帮助

- `git help <verb>`
- `git <verb> --help`
- `man git-<verb>`

## 获取Git仓库

- `git init` 在现有项目中初始化仓库
- `git clone [url]` 克隆现有的仓库

## 记录每次更新到仓库

文件四种状态

![image-20210911144939946](assets/image-20210911144939946.png)

- `git status` 检查当前文件状态

- `git add` 跟踪新文件

- `git status -s` 

  A 标记：新添加到暂存区中的文件

  M标记：出现在靠左边的M 表示该文件被修改了并放入了暂存区，出现在右边的 M 表示该文件被修改了但是还没放入暂存区

