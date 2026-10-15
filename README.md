# wt-cicd-lab 工作流模板

在组织仓库的 **Actions → New workflow** 页面分别添加以下两个模板。

| 模板 | 触发条件 | 执行流程 |
| --- | --- | --- |
| wt-cicd-lab C++ 开发分支构建 | 默认分支以外的分支 push | Prepare → Build Release → Test |
| wt-cicd-lab C++ 默认分支打标与归档 | 默认分支 push | Build Release、Test → 自动 tag → Compile RelWithDebInfo、Test → artifact |

模板中的 `$default-branch` 在创建时由 GitHub 替换为使用方的默认分支名。
例如 `test_cicd_repo` 会替换为 `main`。两个模板都只监听分支 push，推送 tag 不会再次触发它们。
合并 PR 到默认分支产生的 push 会进入打标与归档流程。

## 手动接入

1. 打开目标仓库 **Actions → New workflow**。
2. 选择 **wt-cicd-lab C++ 开发分支构建**，点击 Configure，提交到默认分支。
3. 再选择 **wt-cicd-lab C++ 默认分支打标与归档**，提交到默认分支；这次提交即触发首次打标与归档。
4. 从最新默认分支创建开发分支并 push，验证只运行 Build、Test。
5. 再向默认分支 push，验证 tag 和 artifact；artifact 在对应 Actions 运行页面下载。

已生成过旧 `cpp-ci.yml` 的仓库应删除旧入口，避免重复触发。

## 使用条件

- 仓库提供可执行的 `./build.sh`，支持 `-b -t Release`、`-b -t RelWithDebInfo`，输出到 `build/`。
- 项目通过 CMake 配置 CTest 测试。
- `./build.sh -T` 必须自动生成一个新版本 tag，并将它推送到 origin。
- 组织 Secret `CI_SSH_KEY` 已授权使用方仓库，并且该 SSH key 可以读取仓库、推送 tag。
- 组织 runner group `lab-runners` 已授权使用方仓库；公开仓库需要该组允许 public repositories。
- 有带 `self-hosted, linux, x64, wt-cicd-lab` 标签的在线 runner。

推送 tag 使用 SSH key，因此入口的 `GITHUB_TOKEN` 仅需要 `contents: read`。
上传 workflow artifact 不需要 Contents write。此流程保存构建 artifact，不创建 GitHub Release 页面。

## 代码职责

- `workflow-templates/cpp-dev.yml`：其他分支 push 的入口。
- `workflow-templates/cpp-release.yml`：默认分支 push 的入口。
- 同名 `.properties.json`：GitHub 页面中的模板名称、说明和分类。
- `wwjszz/cicd_template@test/wt-cicd-lab`：实际构建、测试、打标和打包实现。

发布工作流内部有两个 job：第一个完成 Build、Test 和打标；第二个通过 `needs` 等待它，
检出生成的 tag 后构建并上传 artifact。tag 必须指向触发这次运行的提交。
同一提交重新运行时复用已存在的版本 tag，后续新提交继续递增版本。
打标 job 排队执行，避免两个 push 同时分配同一个版本号。

实验项目的首个版本是 `0.1.0`，之后递增 patch，例如 `0.1.1`、`0.1.2`。
artifact 包含完整 `build/` 压缩包、`build-info.txt` 和 `SHA256SUMS`。

修改这里的模板只影响之后创建的工作流，已经生成的 YAML 需要自行更新。
公共实现的实验分支更新后，引用该分支的后续新运行会使用新实现。

官方说明：[创建组织工作流模板](https://docs.github.com/en/actions/how-tos/reuse-automations/create-workflow-templates)。
