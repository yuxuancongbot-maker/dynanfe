# Git 使用教程
> 基于 DynaNFE 项目的实战经验总结

---

## 1. 初始配置

```bash
git config --global user.name "你的名字"
git config --global user.email "你的邮箱"

# 设置默认分支名为 main
git config --global init.defaultBranch main
```

---

## 2. 创建仓库并首次推送

```bash
# 进入项目目录
cd /path/to/your/project

# 初始化
git init
git branch -M main

# 关联远程仓库（在 GitHub 上先创建好空仓库）
git remote add origin https://TOKEN@github.com/用户名/仓库名.git

# 首次提交推送
git add .
git commit -m "first commit"
git push -u origin main
```

---

## 3. .gitignore 配置

在项目根目录创建 `.gitignore`，ML 项目推荐配置：

```gitignore
# Python
__pycache__/
*.pyc
*.pyo

# 虚拟环境
venv/
.env/

# 模型权重（重要！不要上传）
*.pth
*.pt
*.ckpt
*.safetensors
checkpoints/
outputs/

# 数据集
data/
datasets/
*.hdf5
*.h5
*.zarr

# 缓存目录（如 openpi_cache）
openpi_cache/
*_cache/

# 日志
runs/
logs/
wandb/
*.log

# Jupyter
.ipynb_checkpoints/

# 系统文件
.DS_Store
.vscode/
```

如果某个文件已经被 git 追踪，需要先移除：

```bash
git rm --cached -r 文件或目录
git add .gitignore
git commit -m "update gitignore"
```

---

## 4. 处理嵌套 git 仓库（Submodule）

当项目依赖另一个 git 仓库时（如 openpi），直接 `git add` 会警告嵌套仓库。
正确做法是使用 submodule。

### 4.1 如果没有修改过该仓库的代码

```bash
# 直接以 submodule 方式添加
git submodule add https://github.com/原始仓库地址 目录名
git add .gitmodules
git commit -m "add xxx as submodule"
git push
```

### 4.2 如果修改过该仓库的代码（本项目情况）

**第一步：Fork 原始仓库到自己的 GitHub**

进入原始仓库页面，点击右上角 Fork。

**第二步：把本地修改推送到自己的 fork**

```bash
cd 子仓库目录   # 如 cd openpi

# 添加自己的 fork 作为远程
git remote add myfork https://TOKEN@github.com/用户名/仓库名.git

# 推送
git push myfork main
```

**第三步：回到主仓库，以 fork 作为 submodule**

```bash
cd ..   # 回到主仓库

git rm --cached -r -f 子仓库目录
git submodule add https://github.com/用户名/fork仓库名 子仓库目录

git add .gitmodules
git commit -m "add xxx as submodule from personal fork"
git push
```

### 4.3 多层嵌套问题

如果子仓库里还有嵌套的 git 仓库（如 `openpi/onestep_pi/openpi`），同样需要处理：

```bash
cd openpi

# 移除嵌套仓库的追踪
git rm --cached -r -f onestep_pi/openpi

# 加入 gitignore 或改为 submodule
echo "onestep_pi/openpi/" >> .gitignore

git add .
git commit -m "remove nested git repo"
git push myfork main
```

---

## 5. Clone 含 Submodule 的仓库

```bash
# 一次性 clone 并初始化所有 submodule
git clone --recurse-submodules https://github.com/用户名/仓库名.git

# 如果已经 clone 但忘加参数
git submodule update --init --recursive
```

---

## 6. 日常工作流

### 修改主仓库文件

```bash
git add .
git commit -m "docs"
git push
```

### 修改子仓库（如 openpi）文件

```bash
cd openpi
git add .
git commit -m ""
git push myfork main

# 回到主仓库更新 submodule 引用
cd ..
git add openpi
git commit -m "update openpi submodule"
git push
```

---

## 7. GitHub 认证

GitHub 已不支持密码登录，需要使用 Personal Access Token（PAT）。

**生成 Token：**

1. GitHub → 头像 → Settings
2. Developer settings → Personal access tokens → Tokens (classic)
3. Generate new token，勾选 `repo` 权限
4. 复制 token（只显示一次，注意保存）

**使用 Token：**

```bash
# 把 token 嵌入远程地址
git remote set-url origin https://TOKEN@github.com/用户名/仓库名.git
```

> **安全提示**：不要把含 token 的命令发给别人，不要把 token 写进代码。
> 如果 token 泄露，立即去 GitHub 撤销（Settings → Developer settings → 找到对应 token → Delete）。

---

## 8. 常见问题

### error: src refspec main does not match any

原因：还没有 commit，本地没有 main 分支。

```bash
git add .
git commit -m "first commit"
git push -u origin main
```

### fatal: Authentication failed

Token 过期或权限不足，重新生成 token 后：

```bash
git remote set-url origin https://新TOKEN@github.com/用户名/仓库名.git
```

### warning: adding embedded git repository

子目录里有 `.git`，需要用 submodule 处理，参考第 4 节。

### 误提交了大文件或权重文件

```bash
# 从 git 追踪中移除（本地文件保留）
git rm --cached -r 文件路径
echo "文件路径" >> .gitignore
git add .gitignore
git commit -m "remove large file from tracking"
git push
```

---

## 9. 本项目仓库结构

```
dynanfe/                        # 主仓库
├── .gitignore
├── .gitmodules                 # submodule 配置
├── solution.md
├── openpi/                     # submodule → github.com/用户名/openpi
│   ├── src/openpi/
│   ├── docs/                   # 研究文档
│   ├── scripts/                # 训练脚本
│   └── third_party/
│       └── LIBERO-plus/        # submodule → github.com/sylvestf/LIBERO-plus
└── ...
```

Clone 方式：

```bash
git clone --recurse-submodules https://github.com/yuxuancongbot-maker/dynanfe.git
```