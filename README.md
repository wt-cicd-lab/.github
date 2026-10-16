# wt-cicd-lab 工作流模板

在组织仓库的 **Actions → New workflow** 页面添加模板。以下两套 C++ 方案按需选择一套。

## 方案一：自动构建，手动决定发版时机

| 模板 | 触发条件 | 执行流程 |
| --- | --- | --- |
| wt-cicd-lab C++ 全分支构建 | 所有分支（含默认分支）push | Prepare → Build Release → Test |
| wt-cicd-lab C++ 手动打标与归档 | 在默认分支点击 Run workflow | Build Release、Test → 自动 tag → Compile RelWithDebInfo、Test → artifact |

新增或修改 workflow 也会执行普通 Build、Test，但不会自动打 tag。
推送 tag 不会触发全分支构建；该模板明确只监听分支 push。

接入步骤：

1. 打开目标仓库 **Actions → New workflow**，选择 **wt-cicd-lab C++ 全分支构建**，提交到默认分支。
2. 再添加 **wt-cicd-lab C++ 手动打标与归档**，同样提交到默认分支。
3. 日常向开发分支或默认分支 push，只执行 Build、Test。
4. 准备发版时，打开 **Actions → C++ manual tag and artifact → Run workflow**，选择默认分支（例如 main）后运行。
5. 无需填写版本号，项目的 `./build.sh -T` 自动计算并推送 tag。完成后在这次 Actions 运行页面下载 artifact。

手动入口需先存在于默认分支，GitHub 才提供 Run workflow 按钮。
此模板只允许默认分支发版；选择其他分支或 tag 运行时，发版 job 会跳过。
普通构建即使已经通过，手动发版仍会针对本次选中的提交重新构建、测试，通过后才打 tag。

从旧方案切换时，请删除或停用业务仓库中“默认分支 push 自动打标”的旧工作流，
否则它仍会在 main push 时自动生成 tag。旧的开发分支构建入口也应替换，避免重复构建。

## 方案二：默认分支每次 push 自动发版

| 模板 | 触发条件 | 执行流程 |
| --- | --- | --- |
| wt-cicd-lab C++ 开发分支构建 | 默认分支以外的分支 push | Prepare → Build Release → Test |
| wt-cicd-lab C++ 默认分支打标与归档 | 默认分支 push | Build Release、Test → 自动 tag → Compile RelWithDebInfo、Test → artifact |

模板中的 `$default-branch` 在创建时由 GitHub 替换为使用方的默认分支名。
例如 `test_cicd_repo` 会替换为 `main`。这两个模板只监听分支 push，推送 tag 不会再次触发它们。
将打标模板提交到默认分支就会触发首次打标；之后修改 workflow、合并 PR 等产生的默认分支 push 也会触发。

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

- `workflow-templates/cpp-ci.yml`：所有分支 push 的 Build、Test 入口。
- `workflow-templates/cpp-release-manual.yml`：手动打标与归档入口。
- `workflow-templates/cpp-dev.yml`：其他分支 push 的旧入口。
- `workflow-templates/cpp-release.yml`：默认分支 push 的自动打标入口。
- 同名 `.properties.json`：GitHub 页面中的模板名称、说明和分类。
- `wwjszz/cicd_template@test/wt-cicd-lab`：实际构建、测试、打标和打包实现。

两种打标入口共用同一套实现。发布工作流内部有两个 job：
第一个完成 Build、Test 和打标；第二个通过 `needs` 等待它，
检出生成的 tag 后构建并上传 artifact。tag 必须指向触发这次运行的提交。
同一提交重新运行时复用已存在的版本 tag，后续新提交继续递增版本。
打标 job 共用并发组并排队执行，避免自动和手动入口同时分配同一个版本号。

实验项目的首个版本是 `0.1.0`，之后递增 patch，例如 `0.1.1`、`0.1.2`。
artifact 包含完整 `build/` 压缩包、`build-info.txt` 和 `SHA256SUMS`。

修改这里的模板只影响之后创建的工作流，已经生成的 YAML 需要自行更新。
公共实现的实验分支更新后，引用该分支的后续新运行会使用新实现。

官方说明：[创建组织工作流模板](https://docs.github.com/en/actions/how-tos/reuse-automations/create-workflow-templates)、
[手动运行工作流](https://docs.github.com/en/actions/how-tos/manage-workflow-runs/manually-run-a-workflow)。
