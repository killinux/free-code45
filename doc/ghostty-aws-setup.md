# Ghostty (Mac) → AWS SSH + screen 配置

记录一次从 Ghostty 连 AWS、在 screen 里跑 Claude 的完整配置过程。

## 场景

- 本地：Mac + Ghostty（`TERM=xterm-ghostty`）
- 远端：AWS Linux（AlmaLinux 10 / RHEL 系 ncurses）
- 远端会话：screen

问题：远端默认没有 `xterm-ghostty` 的 terminfo，不处理的话 screen/vim/less 颜色和按键会乱。

## 一、安装 xterm-ghostty terminfo 到远端

### 推荐方案：scp Ghostty 自带编译好的 terminfo

Ghostty 的 terminfo 是用 Zig 代码在构建时生成的，**仓库里没有现成的 `.terminfo` 源文件**，不能从 GitHub 直接 curl。但 Mac 上 Ghostty.app 内已经有编译产物，直接 scp。

在 **Mac 本地** 执行：

```bash
scp /Applications/Ghostty.app/Contents/Resources/terminfo/78/xterm-ghostty \
    root@<AWS-IP>:~/.terminfo/78/
```

如果报：

```
scp: Received message too long 1751476325
scp: Ensure the remote shell produces no output for non-interactive sessions
```

说明远端 `.bashrc`/`.zshrc` 在非交互 shell 里也输出了东西，见下节修复。

### 替代方案：从 Mac pipe terminfo 到远端

```bash
infocmp -x xterm-ghostty | ssh <aws> 'tic -x -'
```

### 处理 ncurses 目录布局差异

Mac 用 hex 目录 (`78/`)，多数 Linux 用字母目录 (`x/`)。把传过去的文件用软链补上字母目录：

```bash
# 在 AWS 上
mkdir -p ~/.terminfo/x
ln -sf ../78/xterm-ghostty ~/.terminfo/x/xterm-ghostty
```

验证：

```bash
TERM=xterm-ghostty infocmp | head -1
# 输出 "Reconstructed via infocmp from file: ..." 即成功
```

## 二、修复 .bashrc 非交互输出污染

scp/rsync 用的是非交互 SSH 会话，如果 `.bashrc` 里有无条件 `echo`，会污染协议。

**原始（坏）**：

```bash
echo "hehe ,I am .bashrc"
```

**修好**：

```bash
[[ $- == *i* ]] && echo "hehe ,I am .bashrc"
```

`$-` 里含 `i` 才是交互式 shell。

验证（应为空输出）：

```bash
bash -c 'true'
```

## 三、自适应 TERM 回退

不要硬编码 `export TERM=xterm-256color`（这样会丢失 Ghostty 原生特性）。改成自适应：

```bash
# ~/.bashrc 末尾
infocmp "$TERM" >/dev/null 2>&1 || export TERM=xterm-256color
```

效果：
- terminfo 已装（xterm-ghostty）→ 保留，享受原生特性
- 未装 → 自动退化到 xterm-256color，不影响使用
- 从 iTerm2 连（TERM=xterm-256color）→ 本身就识别，无操作

## 四、.screenrc

```
termcapinfo xterm* ti@:te@

# screen 内部固定 256 色，兼容性最好
term screen-256color

# 256 色与背景色擦除
attrcolor b ".I"
defbce on

# 滚屏历史
defscrollback 10000
```

说明：外层 Ghostty 跑 `xterm-ghostty`，进 screen 后 `$TERM` 会切成 `screen-256color`，vim/less 都走这个。

## 五、验证完整链路

```bash
# 1) 从 Ghostty 新开 SSH
echo $TERM          # 期望：xterm-ghostty

# 2) 进 screen
screen
echo $TERM          # 期望：screen-256color

# 3) 256 色测试
for i in {0..255}; do printf "\x1b[38;5;${i}m%3d " "$i"; done; echo
```

## 六、Ghostty vs iTerm2 对比（选型参考）

Ghostty 强在启动快、GPU 渲染、配置简单。但长期 SSH + tmux/screen 工作流，iTerm2 目前仍更成熟：

- **tmux -CC 原生集成** —— tmux 窗口变成 iTerm2 原生 tab
- **Shell Integration** —— `cmd+↑/↓` 跳提示符，命令成功/失败标记
- **Profile 按 host 自动切**
- **Triggers / Smart Selection**
- **Instant Replay**
- **Sixel / 图片**支持更成熟

建议：日常重度 SSH 回 iTerm2 + tmux；Ghostty 留给本地快速终端。
