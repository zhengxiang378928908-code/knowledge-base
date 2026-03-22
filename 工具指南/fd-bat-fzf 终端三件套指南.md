# fd / bat / fzf 终端三件套指南

> 三个工具分别替代 `find`、`cat`、模糊搜索，组合使用效率极高。

## 一、fd — 更快的 find

```bash
# 安装
brew install fd

# 基础用法：按文件名搜索（默认递归、忽略 .gitignore）
fd readme              # 搜索文件名包含 readme 的文件
fd -e md               # 搜索所有 .md 文件
fd -e js -e ts         # 搜索所有 .js 和 .ts 文件

# 指定目录搜索
fd "\.py$" src/        # 在 src/ 下搜索 .py 文件

# 按类型过滤
fd -t f config         # 只搜文件（f=file, d=directory, l=symlink）
fd -t d node_modules   # 只搜目录

# 包含隐藏文件 / 忽略文件
fd -H .env             # -H 搜索隐藏文件
fd -I node_modules     # -I 不遵守 .gitignore

# 排除目录
fd -e ts --exclude node_modules

# 搜到后执行命令
fd -e log -x rm {}                    # 删除所有 .log 文件
fd -e png -x convert {} {.}.webp      # 批量转换图片格式
fd -e tmp -X rm                       # -X 把结果合并成一条命令执行

# 限制深度
fd -d 2 -e md          # 最多搜 2 层
```

### fd 实用场景

```bash
# 快速定位配置文件
fd -H -g "*.config.*"

# 查找大于 10M 的文件
fd -t f -S +10m

# 查找最近 1 天内修改的文件
fd -t f --changed-within 1d

# 结合 xargs 批量操作
fd -e java | xargs wc -l       # 统计所有 Java 文件行数
```

---

## 二、bat — 更好的 cat

```bash
# 安装
brew install bat

# 基础用法：带语法高亮 + 行号查看文件
bat file.py
bat src/*.ts            # 同时查看多个文件

# 指定语言高亮（文件无后缀时）
bat -l json config
bat -l sql < query.txt

# 只显示行号，不要装饰
bat -p file.py          # plain 模式，去掉边框和行号
bat -pp file.py         # 纯文本输出（适合管道）

# 显示指定行范围
bat -r 10:20 file.py    # 只看第 10-20 行
bat -r :50 file.py      # 前 50 行
bat -r 100: file.py     # 第 100 行到末尾

# 高亮特定行
bat -H 15 file.py       # 高亮第 15 行
bat -H 10:20 file.py    # 高亮 10-20 行

# 显示不可见字符
bat -A file.txt         # 显示 tab、换行等

# 查看 diff
bat --diff file.py      # 显示 git diff 标记

# 作为 man 的 pager
export MANPAGER="sh -c 'col -bx | bat -l man -p'"

# 主题
bat --list-themes       # 查看所有主题
bat --theme="Dracula" file.py
```

### bat 配置文件

```bash
# 创建配置文件
mkdir -p ~/.config/bat && bat --generate-config-file

# ~/.config/bat/config 常用配置
--theme="Catppuccin Mocha"
--style="numbers,changes,grid"
--italic-text=always
--map-syntax "*.conf:INI"
```

---

## 三、fzf — 模糊搜索神器

```bash
# 安装
brew install fzf

# 基础用法：交互式模糊搜索
fzf                     # 搜索当前目录所有文件
vim $(fzf)              # 选中文件后用 vim 打开

# 从管道输入搜索
cat ~/.zsh_history | fzf
ps aux | fzf            # 搜索进程
brew list | fzf         # 搜索已安装的包

# 预览文件内容
fzf --preview 'bat --color=always {}'

# 多选模式
fzf -m                  # Tab 多选，Enter 确认
fzf -m | xargs rm       # 多选后删除
```

### fzf 快捷键（需要安装 shell 集成）

```bash
# 安装 shell 集成（zsh）
# 在 .zshrc 中添加:
source <(fzf --zsh)

# 快捷键
Ctrl+T    # 搜索文件，粘贴路径到命令行
Ctrl+R    # 模糊搜索历史命令
Alt+C     # 搜索目录并 cd 进入（macOS 用 ESC+C）
```

### fzf 高级用法

```bash
# 指定搜索命令（用 fd 替代默认的 find）
export FZF_DEFAULT_COMMAND='fd --type f --hidden --follow --exclude .git'
export FZF_CTRL_T_COMMAND="$FZF_DEFAULT_COMMAND"
export FZF_ALT_C_COMMAND='fd --type d --hidden --follow --exclude .git'

# 自定义预览窗口
export FZF_DEFAULT_OPTS="
  --height 60%
  --layout=reverse
  --border
  --preview 'bat --color=always --style=numbers --line-range=:200 {}'
  --preview-window=right:60%
"

# 搜索文件内容（grep + fzf）
rg --line-number . | fzf --delimiter : --preview 'bat --color=always --highlight-line {2} {1}'

# Git 分支切换
git branch | fzf | xargs git checkout

# Git log 浏览
git log --oneline | fzf --preview 'git show {1}'

# 杀进程
kill -9 $(ps aux | fzf | awk '{print $2}')

# Docker 容器管理
docker ps -a | fzf | awk '{print $1}' | xargs docker rm
```

---

## 四、三件套组合技

```bash
# fd + fzf + bat：搜文件 → 模糊选择 → 预览并打开
fd -e py | fzf --preview 'bat --color=always {}' | xargs code

# fd + fzf：快速 cd 到目录
cd $(fd -t d | fzf)

# 交互式搜索并编辑（最常用）
vim $(fd -t f | fzf --preview 'bat --color=always --line-range=:100 {}')

# 搜索 + 预览 + 多选删除
fd -e log | fzf -m --preview 'bat --color=always {}' | xargs rm

# 交互式 git add
git ls-files -m | fzf -m --preview 'bat --color=always --diff {}' | xargs git add
```

---

## 五、推荐 .zshrc 配置

```bash
# ---- fd / bat / fzf 配置 ----

# fd 作为 fzf 的默认搜索引擎
export FZF_DEFAULT_COMMAND='fd --type f --hidden --follow --exclude .git'
export FZF_CTRL_T_COMMAND="$FZF_DEFAULT_COMMAND"
export FZF_ALT_C_COMMAND='fd --type d --hidden --follow --exclude .git'

# fzf 默认选项（带 bat 预览）
export FZF_DEFAULT_OPTS="
  --height 60%
  --layout=reverse
  --border
  --preview 'bat --color=always --style=numbers --line-range=:200 {}'
"

# bat 别名
alias cat='bat -pp'         # 替代 cat（纯文本输出）
alias catn='bat'            # 带行号和语法高亮
alias less='bat --paging=always'

# 常用函数
# 模糊搜索并用编辑器打开
fe() {
  local file
  file=$(fd -t f | fzf --preview 'bat --color=always {}') && ${EDITOR:-vim} "$file"
}

# 模糊搜索目录并 cd
fcd() {
  local dir
  dir=$(fd -t d | fzf --preview 'ls -la {}') && cd "$dir"
}

# 模糊搜索历史命令
fh() {
  eval $(history | fzf --tac | awk '{$1=""; print}')
}

# fzf shell 集成
source <(fzf --zsh)
```
