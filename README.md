# 26/5/9 CI/CD 经验总结

## 1. 架构总览：声明式 CI/CD 流水线

本项目基于 GitHub Actions 构建，采用声明式 YAML 配置文件，实现持续集成（CI）与持续交付/部署（CD）的职责分离。所有工作流文件必须存放于严格的系统级目录：`.github/workflows/`。

### 1.1 持续集成门禁 (Continuous Integration - `test.yml`)
* **核心职责**：代码质量门禁（Code Gating），防止存在缺陷的代码合并至主干。
* **触发机制**：监听对 `main` 分支的 `push` 与 `pull_request` 事件。
* **执行链路**：检出代码库 -> 挂载隔离的 Docker 容器环境 -> 构建虚拟环境 (`uv sync`) -> 执行单元测试 (`pytest`) -> 阻断或放行。

### 1.2 制品发布引擎 (Continuous Deployment - `deploy.yml`)
* **核心职责**：制品分发（Artifact Distribution）与版本固化。
* **触发机制**：监听以 `v` 开头的轻量/附注标签（Tag）推送事件（Glob 匹配 `v*`）。
* **执行链路**：检出对应 Tag 代码 -> 环境构建 -> 调用 GitHub REST API 创建 Release 页面 -> 自动生成变更日志（Changelog）。

---

## 2. 避坑与故障排查记录

### 2.1 仓库初始化冲突 (Git Non-Fast-Forward Error)
* **故障现象**：执行 `git push` 时被远端拒绝，报错 `! [rejected] main -> main (non-fast-forward)`。
* **根本原因**：在 GitHub 网页端创建仓库时勾选了生成 `README.md` 或 `.gitignore`，导致远端产生独立 Commit 历史，与本地未同步的历史发生冲突。
* **标准规范**：在本地已有代码记录时，远端必须初始化为 **100% 纯空仓库**，随后通过本地推送建立全量拓扑树。

### 2.2 工作区状态不同步 (Everything up-to-date)
* **故障现象**：修改文件后直接 `git push`，提示已是最新状态，远端代码未更新。
* **根本原因**：Git 缺乏对修改文件的自动暂存（Stage）机制。未执行 `add` 和 `commit` 指令，导致推送载荷（Payload）为空。
* **标准规范**：遇到状态异常，首选 `git status` 检查工作树。遵循标准提交范式：
    ```bash
    git add .                            # 暂存所有变更
    git commit -m "chore: update files"  # 提交至本地历史
    git push origin main                 # 同步至远端
    ```

### 2.3 容器镜像拉取失败 (Registry Access Denied)
* **故障现象**：流水线在初始化环境时中断，报错 `pull access denied for astralapp/uv`。
* **根本原因**：第三方工具库（如 `uv`）的 Docker 镜像注册表（Registry）地址变更，旧命名空间被废弃或转为私有。
* **修复方案**：跟进官方文档变更，将容器编排地址更新为最新的 Github Container Registry (GHCR) 路径：
    ```yaml
    # 修正前：container: astralapp/uv:python3.12-bookworm-slim
    container: ghcr.io/astral-sh/uv:python3.12-bookworm-slim
    ```

### 2.4 测试模块解析失败 (ModuleNotFoundError)
* **故障现象**：Pytest 运行中断，抛出 `ModuleNotFoundError: No module named 'main'`。
* **根本原因**：Python 默认的模块寻址机制（`sys.path`）未能将项目根目录包含在内。测试引擎在 `tests/` 子目录下无法跨级检索业务逻辑代码。
* **修复方案**：修改项目元数据配置文件 `pyproject.toml`，显式注入环境变量（`PYTHONPATH`），修正解析基准路径：
    ```toml
    [tool.pytest.ini_options]
    pythonpath = ["."]
    ```

### 2.5 远端 API 鉴权阻断 (Permission Denied for Release)
* **故障现象**：`deploy.yml` 运行至最后一步 `Create GitHub Release` 时抛出 403 权限异常。
* **根本原因**：遵循最小权限原则，GitHub Actions 分配的缺省 `GITHUB_TOKEN` 仅具备仓库读权限（Read-only），无权写入 Release 数据资产。
* **修复方案**：在 Workflow 或 Job 层级实施显式的权限提权（Token Scoping）：
    ```yaml
    permissions:
      contents: write
    ```

---

## 3. 版本控制规范 (Versioning Policy)
在触发 CD 流水线时，严禁复用已存在的版本标签。Git 标签具有不可变性（Immutability）。必须遵循 **语义化版本规范 (SemVer)** 递增标签推送：
* **Patch (v1.0.x)**：向下兼容的缺陷修复（Bug fixes）。
* **Minor (v1.x.0)**：向下兼容的新功能追加（Features）。
* **Major (vX.0.0)**：破坏性变更或重构（Breaking changes）。

**触发指令示例：**
```bash
git tag v1.1.0
git push origin v1.1.0