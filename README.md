# C++ CI 使用说明

使用前准备好：

- 仓库或组织 Secret `CI_SSH_KEY`，用于拉取源码、私有依赖和推送 tag，并授权业务仓库使用。
- 在线 runner，标签包含 `self-hosted, linux, x64, wwj-cicd`；提供 Docker 和镜像 `localhost:5000/cxx_builder_devtoolset_8:1.3`。
- 可执行的项目 `./build.sh`，支持 `-b -t Release`、`-b -t Debug -C`、`-b -t RelWithDebInfo`、`-T`。构建输出到 `build/`，测试程序放在 `build/run_test/`；`-T` 负责创建并推送新 tag。

## 1. 使用三个模板

| 模板文件 | GitHub 页面名称 | 触发条件 | 执行内容 |
| --- | --- | --- | --- |
| `cpp-build.yml` | C++ 构建与测试（非 master） | 非 `master` 分支 push | Release 构建、测试 |
| `cpp-coverage.yml` | C++ 覆盖率（非 master） | 非 `master` 分支 push | Debug 构建、测试、上传覆盖率报告 |
| `cpp-release.yml` | C++ 打标与归档（仅 master） | 仅 `master` 分支 push | 构建测试、打 tag、RelWithDebInfo 构建测试、上传归档 |

1. 在业务仓库打开 **Actions → New workflow**，搜索上面的模板名称，点击 **Configure**。
2. 按需添加模板，将生成的 YAML 提交到业务仓库的 `.github/workflows/`。
3. 向开发分支 push 运行 build、coverage；向 `master` push 运行 release。

也可以直接复制本仓库 [`workflow-templates/`](workflow-templates/) 下的三个 `.yml` 到业务仓库的 `.github/workflows/`，不需要复制 `.properties.json`。

三个模板都忽略 tag push。build、coverage 不能在 `master` 运行；release 只能在 `master` 运行。
覆盖率报告和发布归档在对应 Actions 运行页面的 **Artifacts** 下载；release 不创建 GitHub Release 页面。

## 2. 自己写 workflow、引用和传参

### 使用公共 workflow

三个固定工作流 `cpp-build.yml`、`cpp-coverage.yml`、`cpp-release.yml` 只接收 Secret `CI_SSH_KEY`，没有可通过 `with` 修改的输入参数。

需要自选 runner、构建类型、测试和覆盖率时，引用 [`cpp-ci.yml`](.github/workflows/cpp-ci.yml)。在业务仓库创建 `.github/workflows/custom-ci.yml`：

```yaml
name: Custom C++ CI

on:
  push:
    branches-ignore: [master]
    tags-ignore: ['**']
  workflow_dispatch:

permissions:
  contents: read

jobs:
  ci:
    uses: wt-cicd-lab/.github/.github/workflows/cpp-ci.yml@com_dev
    with:
      runner-label: wwj-cicd
      build-type: Debug
      run-tests: true
      coverage: true
    secrets:
      CI_SSH_KEY: ${{ secrets.CI_SSH_KEY }}
```

| 参数 | 类型 | 必填 / 默认值 | 说明 |
| --- | --- | --- | --- |
| `runner-label` | string | 必填 | runner 的额外标签，示例为 `wwj-cicd` |
| `build-type` | string | `Release` | `Debug`、`Release`、`RelWithDebInfo` |
| `run-tests` | boolean | `true` | 是否运行测试 |
| `coverage` | boolean | `false` | 是否开启覆盖率构建并上传报告 |

例如普通构建使用 `build-type: Release`、`coverage: false`；覆盖率构建使用 `build-type: Debug`、`coverage: true`。
`coverage: true` 时始终运行测试，即使 `run-tests: false`。
普通参数放在 `with`，密钥放在 `secrets`；boolean 参数使用 `true` / `false`，不加引号。
`cpp-ci.yml` 的触发分支由调用方的 `on` 决定，不包含打标和发布归档。

### 自己编排每个步骤

如果需要自己安排阶段，在 job 的 `steps` 中引用 [`actions/`](actions/)。先完成源码 checkout 和 SSH agent 启动，具体配置可参考 [`cpp-ci.yml`](.github/workflows/cpp-ci.yml)，然后调用：

```yaml
- name: Prepare
  uses: wt-cicd-lab/.github/actions@com_dev
  with:
    phase: prepare

- name: Build
  uses: wt-cicd-lab/.github/actions@com_dev
  with:
    phase: build
    build-type: Debug
    coverage: "true"
```

Action 参数：`phase` 必填；`build-type` 默认 `Release`；`coverage` 默认 `"false"`，传字符串 `"true"` / `"false"`。
构建后可继续调用 `phase: test`、`phase: coverage`；完整打标与归档流程参考 [`cpp-release.yml`](.github/workflows/cpp-release.yml)。

公共 workflow 在 `jobs.<job>.uses` 中引用，Action 在 `steps[].uses` 中引用。当前模板和示例均使用已发布的 `@com_dev`，它是公共实现的分支名，与业务仓库的 `master` 规则无关。

更多语法见 [GitHub 复用工作流说明](https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows)。
