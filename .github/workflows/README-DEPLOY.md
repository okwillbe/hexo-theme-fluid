# GitHub Actions 部署配置说明

## 问题原因

原 `deploy-theme.yml` 初始化失败的主要原因:

1. **缺少依赖的 workflow**: 引用了不存在的 `verify.yml` workflow
2. **文件编码问题**: 中文注释显示为乱码,文件编码不是标准 UTF-8
3. **逻辑缺陷**: 在非 release 事件时无法获取构建产物

## 修复内容

### 1. 新增独立的 build 任务
- 替代了缺失的 `verify.yml`
- 包含完整的构建流程: checkout → 安装依赖 → 构建 → 上传产物
- 使用 `actions/upload-artifact@v4` 保存构建结果

### 2. 优化 deploy 任务
- 通过 `needs: build` 确保构建完成后再部署
- 使用 `actions/download-artifact@v4` 下载构建产物
- 更新所有 actions 到最新版本 (v4)

### 3. 添加安全机制
- SSH 部署添加条件判断 `if: vars.ENABLE_SSH_DEPLOY == 'true'`
- 避免缺少 secrets 配置时导致整个 workflow 失败

## 后续配置步骤

### 方案 A: 启用 SSH 部署(需要远程服务器)

如果你需要自动部署到远程服务器,需要配置以下内容:

#### 1. 在 GitHub 仓库设置中添加 Secrets
进入 `Settings` → `Secrets and variables` → `Actions` → `New repository secret`:

- `REMOTE_HOST`: 远程服务器地址 (例如: `123.456.789.0`)
- `REMOTE_USER`: SSH 用户名 (例如: `root` 或 `hexo`)
- `SSH_PRIVATE_KEY`: SSH 私钥内容 (完整的私钥,包括 `-----BEGIN ... KEY-----` 部分)

#### 2. 添加 Variables
进入 `Settings` → `Secrets and variables` → `Actions` → `Variables` → `New repository variable`:

- Name: `ENABLE_SSH_DEPLOY`
- Value: `true`

### 方案 B: 仅构建和发布(推荐初期使用)

如果暂时不需要自动部署到服务器:

1. **不需要配置任何 secrets**
2. workflow 会自动:
   - 在每次 push 到 master/okwillbe-patch 分支时构建
   - 在创建 release 时构建并上传压缩包到 release 资产
3. 你可以手动下载 release 中的 `fluid-*.tar.gz` 部署

## 自定义构建步骤

当前构建步骤是通用模板,请根据你的项目实际情况调整 `deploy-theme.yml` 第 30-38 行:

```yaml
- name: Build Theme
  run: |
    # 如果有 npm 构建脚本,使用:
    npm run build
    
    # 或者如果只需要复制文件:
    mkdir -p dist
    cp -r ./* dist/
    rm -rf dist/.git dist/.github dist/node_modules
```

## 测试 Workflow

### 方法 1: 手动触发
1. 进入 GitHub 仓库的 `Actions` 标签
2. 选择 `Deploy Hexo Theme`
3. 点击 `Run workflow` → `Run workflow`

### 方法 2: 推送代码
```bash
git add .github/workflows/deploy-theme.yml
git commit -m "fix: 修复 GitHub Actions 部署配置"
git push origin master
```

### 方法 3: 创建 Release
1. 进入 GitHub 仓库的 `Releases`
2. 点击 `Draft a new release`
3. 创建新标签并发布

## 验证成功标志

workflow 运行成功后,你应该看到:

1. ✅ `build` 任务完成,产生 `verified-build` 产物
2. ✅ `deploy` 任务完成
3. ✅ (如果是 release) 在 release 页面看到 `fluid-*.tar.gz` 文件

## 故障排查

### 如果 build 失败
- 检查项目是否有 `package.json`
- 检查 Node.js 版本是否兼容 (当前配置为 18)
- 查看构建日志中的具体错误信息

### 如果 deploy 失败
- 确认 secrets 配置正确
- 检查 SSH 私钥格式 (必须是完整的私钥,包括头尾)
- 验证远程服务器路径 `/home/hexo/blog` 是否存在

## 备份文件

原始文件已备份为: `deploy-theme-backup.yml`

如需恢复,运行:
```bash
cd .github/workflows
mv deploy-theme.yml deploy-theme-new.yml
mv deploy-theme-backup.yml deploy-theme.yml
```
