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

在 job 的 `steps` 中依次调用 [`setup`](actions/setup/action.yml) 和 [`build-test`](actions/build-test/action.yml)：

```yaml
- name: Setup
  uses: wt-cicd-lab/.github/actions/setup@com_dev
  with:
    ssh-key: ${{ secrets.CI_SSH_KEY }}

- name: Build and test
  uses: wt-cicd-lab/.github/actions/build-test@com_dev
  with:
    build-type: Debug
    run-tests: "true"
    coverage: "false"
```

`setup` 完成源码 checkout 和 SSH agent 初始化：`ssh-key` 必填，`ref` 可选，留空时检出触发事件对应的源码。
`build-test` 依次执行 Prepare、Build、Test：`build-type` 默认 `Release`，`run-tests` 默认 `"true"`，`coverage` 默认 `"false"`。Action 的开关参数使用字符串 `"true"` / `"false"`；开启 coverage 时始终运行测试。

`build-test` 的 `coverage: "true"` 只开启插桩并运行测试。生成报告还需调用 [`actions/`](actions/action.yml)，传入 `phase: coverage`，再上传 `build/test_coverage/`；完整步骤参考 [`cpp-coverage.yml`](.github/workflows/cpp-coverage.yml)。
单个阶段也可直接调用 `actions/`：`phase` 必填，支持 `prepare`、`build`、`test`、`coverage`、`tag`、`package`；`build-type` 和 `coverage` 的默认值分别为 `Release`、`"false"`。打标与归档的完整流程参考 [`cpp-release.yml`](.github/workflows/cpp-release.yml)。

公共 workflow 在 `jobs.<job>.uses` 中引用，Action 在 `steps[].uses` 中引用。当前模板和示例均使用已发布的 `@com_dev`，它是公共实现的分支名，与业务仓库的 `master` 规则无关。
公共实现内部使用 `$/actions/...` 引用同仓库、同一提交的 Action，业务仓库无需复制这些文件。

更多语法见 [GitHub 复用工作流说明](https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows)、[同仓库 Action 引用说明](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#jobsjob_idstepsuses)。
