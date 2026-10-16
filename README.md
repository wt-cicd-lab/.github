# C++ CI 工作流与组织模板

为业务仓库提供三套入口，分支规则固定使用 `master`，不随仓库默认分支变化。

| 模板 | 触发条件 | 执行内容 |
| --- | --- | --- |
| `cpp-build.yml` | 非 `master` 分支 push | Prepare → Release 构建 → 测试 |
| `cpp-coverage.yml` | 非 `master` 分支 push | Prepare → Debug 覆盖率构建 → 测试 → 覆盖率报告与上传 |
| `cpp-release.yml` | 仅 `master` 分支 push | Release 构建与测试 → 自动打 tag → 检出该 tag → RelWithDebInfo 构建与测试 → 打包上传 |

三个模板均不响应 tag push。分支删除事件也会跳过。
例如向 `feature/demo` 或 `main` push 会执行 build、coverage；向 `master` push 只执行 release。
release 在模板入口和公共工作流内部都检查 `refs/heads/master`，其他分支直接调用公共工作流也会跳过。
build、coverage 的公共工作流同样限制为非 `master` 分支的 push。

## 目录与命名

```text
.github/workflows/
  cpp-build.yml          # 可复用：Release 构建与测试
  cpp-coverage.yml       # 可复用：Debug 构建、测试与覆盖率
  cpp-release.yml        # 可复用：打标及构建产物归档
  cpp-ci.yml             # 保留的通用可复用入口，三个模板不引用它
actions/
  action.yml            # 执行指定阶段的 composite action
  run.sh                # 启动构建容器
  scripts/              # prepare/build/test/coverage/tag/package 实现
workflow-templates/
  cpp-build.yml
  cpp-build.properties.json
  cpp-coverage.yml
  cpp-coverage.properties.json
  cpp-release.yml
  cpp-release.properties.json
```

`cpp-build.yml`、`cpp-coverage.yml`、`cpp-release.yml` 的名字可以保留，分别与三种职责对应。
GitHub Actions 页面显示的名称由 YAML 中的 `name` 决定。
`cpp-release.yml` 当前负责打标和 artifact 归档，不创建 GitHub Release 页面。

本仓库名称是 `wt-cicd-lab/.github`；公共工作流仍必须放在仓库内部的 `.github/workflows/`。
因此服务器路径是 `/home/wangwenjie/test_dev/.github/.github/workflows/`，前一个 `.github` 是仓库目录。

## 接入与发布

1. 将公共工作流和 `actions/` 发布到本仓库的 `com_dev` 分支。当前三个模板和公共 action 引用均固定使用 `@com_dev`。
2. 将 `workflow-templates/` 发布到组织 `.github` 仓库的默认分支，模板才会在业务仓库的 **Actions → New workflow** 中出现。
3. 在业务仓库按需添加 build、coverage 和 release 模板，保存到业务仓库的 `.github/workflows/`。
4. 向非 `master` 分支 push 运行 build、coverage；合并或 push 到 `master` 自动运行 release。

模板中的调用路径示例：

```yaml
uses: wt-cicd-lab/.github/.github/workflows/cpp-build.yml@com_dev
```

公共工作流中的 action 引用为 `wt-cicd-lab/.github/actions@com_dev`。
两者都指向本仓库；业务项目无需复制 `actions/`。
`com_dev` 是公共实现仓库的引用分支，与业务仓库中允许发版的 `master` 分支规则相互独立。
以后更换公共实现引用时，需要同时更新模板中的 workflow 引用与公共工作流中的 action 引用。

模板更新只影响之后创建的业务工作流，已有业务仓库中的 YAML 需要自行同步。

## 运行条件与产物

- 业务仓库可读取组织 Secret `CI_SSH_KEY`；该 key 能读取源码及私有依赖，并有推送 tag 的权限。
- 业务仓库有可使用的在线 runner，标签包含 `self-hosted, linux, x64, wwj-cicd`。
- runner 提供 Docker，能够使用 `localhost:5000/cxx_builder_devtoolset_8:1.3` 镜像。
- 镜像提供 devtoolset-8、`git_setup`、lcov、genhtml 和 lcov_cobertura 等脚本依赖。
- 项目提供可执行的 `./build.sh`，支持 `-b -t Release`、`-b -t Debug -C`、`-b -t RelWithDebInfo` 和 `-T`；`-T` 自动创建并推送一个新版本 tag。
- 构建产物输出到 `build/`，测试可执行文件输出到 `build/run_test/`。

coverage 上传 `build/test_coverage/`，其中包含 HTML、LCOV 和 Cobertura 报告。
release 上传 `artifacts/build-output.tar.gz`、`build-info.txt` 和 `SHA256SUMS`。
归档前会校验构建类型、coverage 开关，以及检出的提交、触发提交和 tag 指向一致。
入口的 `GITHUB_TOKEN` 只需要 `contents: read`，推送 tag 使用 SSH key。

官方说明：[创建组织工作流模板](https://docs.github.com/en/actions/how-tos/reuse-automations/create-workflow-templates)、[复用工作流](https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows)。
