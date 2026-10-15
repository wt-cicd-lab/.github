# wt-cicd-lab 工作流模板

本仓库提供组织内各仓库在 **Actions → New workflow** 页面选择的工作流模板。

## 从模板创建工作流

1. 打开目标仓库的 Actions 页面。如果已有运行记录，点击 New workflow。
2. 在组织模板区域找到 **wt-cicd-lab C++ 构建与测试**，点击 Configure。
3. 检查生成在 `.github/workflows/` 下的 YAML，然后提交到目标仓库的默认分支。
4. 提交会触发第一次构建；也可以在工作流页面点击 Run workflow。

模板中的 `$default-branch` 会在创建时替换为目标仓库的默认分支名。
例如，`test_cicd_repo` 生成的是 `main`，使用 `master` 的仓库则生成 `master`。

触发条件：

- push 到默认分支；
- 向默认分支创建或更新 Pull Request；
- workflow_dispatch 手动触发。

## 使用条件

- 仓库提供可执行的 `./build.sh`，支持 `-b -t Release`，输出构建目录 `build/`。
- 项目通过 CMake 配置 CTest 测试。
- 组织 Secret `CI_SSH_KEY` 已授权目标仓库使用，并且对应 SSH key 可以读取该仓库。
- 组织 runner group `lab-runners` 已允许目标仓库使用；公开仓库需要该组允许 public repositories。
- 有带 `self-hosted, linux, x64, wt-cicd-lab` 标签的在线 runner。

当前模板调用：

```yaml
uses: wwjszz/cicd_template/.github/workflows/cpp-build.yml@test/wt-cicd-lab
```

该可复用工作流负责 Prepare、Build Release、Test。编译直接在实验 runner 内执行。

## 文件职责

- `workflow-templates/cpp-ci.yml`：复制到使用方仓库的工作流入口。
- `workflow-templates/cpp-ci.properties.json`：模板在 GitHub 页面中的名称、说明和分类。
- `wwjszz/cicd_template` 的实验分支：实际执行构建与测试的公共实现。

修改本仓库的模板只影响之后创建的工作流，不会自动修改已经生成的 YAML。
已生成的工作流仍会按照它的 `uses` 引用调用公共实现；当前引用的实验分支更新后，
后续新运行会使用该分支的新实现。

官方说明：[创建组织工作流模板](https://docs.github.com/en/actions/how-tos/reuse-automations/create-workflow-templates)。
