# 博客项目记忆

## 项目信息
- 项目：Hexo 博客
- 仓库：https://github.com/LoyalToOrigin/loyaltoorigin.github.io
- 网站：https://loyaltoorigin.github.io
- 源码分支：hexo_source
- 部署分支：master

## 技术栈
- Hexo: 8.1.1
- Node.js: 16.20.2（通过 .nvmrc 锁定）
- 渲染器：hexo-renderer-marked, hexo-renderer-ejs, hexo-renderer-stylus
- 部署方式：SSH (git@github.com)

## 部署命令
```bash
export NVM_DIR="$HOME/.nvm"
source "$NVM_DIR/nvm.sh"
nvm use 16
npx hexo clean && npx hexo g && npx hexo d
```

## 网络配置（关键）
由于 DNS 污染，`~/.ssh/config` 配置了 GitHub 的直接 IP：
```ssh
Host github.com
    HostName 20.205.243.166
    User git
    Port 22
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
```

## npm 配置
- 镜像源：https://registry.npmmirror.com（淘宝镜像）

## 重要文件
- `.nvmrc`: 锁定 Node.js 16 版本
- `.deploy_gitignore`: 防止 `.deploy_git` 目录被包含进部署
- `_config.yml`: Hexo 配置，deploy.repo 使用 SSH 方式

## 已知问题
- GitHub Pages 有 25 个依赖漏洞（3 严重、12 高、9 中、1 低），需要更新依赖包
