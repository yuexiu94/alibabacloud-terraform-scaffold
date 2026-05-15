# GitHub 集成方案 - 直连 IacService 模式

基于 GitHub Actions 与阿里云自动化服务台（IacService）的 CI/CD 集成方案，通过 IacService API 直接触发 Terraform 资源栈的自动化部署。

> **🌐 语言**：[中文文档](README-CN.md) | [English Docs](README.md)

## 目录

- [概述](#概述)
- [前期准备](#前期准备)
  - [准备代码仓库](#准备代码仓库)
  - [初始化云端基础设施](#初始化云端基础设施)
  - [GitHub 配置](#github-配置)
- [GitHub Actions 工作流配置](#github-actions-工作流配置)
  - [工作流文件](#工作流文件)
  - [工作流依赖关系](#工作流依赖关系)
- [日常使用](#日常使用)
  - [创建自动化服务台资源栈](#创建自动化服务台资源栈)
  - [配置保密参数](#配置保密参数)
  - [变更流程](#变更流程)
  - [运行参数说明](#运行参数说明)
- [运维参考](#运维参考)
  - [故障排查](#故障排查)
  - [CI 依赖](#ci-依赖)
  - [多环境配置](#多环境配置)
- [附录](#附录)
  - [工作流设计详解](#工作流设计详解)

---

# 概述

GitHub 集成的核心优势在于：
- 利用 GitHub Actions 作为 CI/CD 引擎
- 通过 Workflow 文件定义自动化流程
- 与 GitHub 原生的 Pull Request、Code Review、Branch Protection 等协作功能深度结合
- 实现"代码审查通过即自动触发部署"的完整闭环

```
Developer → PR Comment → GitHub Actions → IacService API(上传代码 + 触发执行) → 执行部署 → IacService API(查询结果) → GitHub Actions → PR Comment
```

上图为从 GitHub 出发管理自动化服务台资源栈的完整运行链路，GitHub Actions 到自动化服务台的交互通过工作流完成代码文件的上传、任务触发以及运行日志的读取。

工作流通过 PR 评论触发，支持以下命令：

```
iac terraform plan [-profile=<profile>] [-stack=<stack>]
iac terraform apply [-profile=<profile>] [-stack=<stack>]
```

> **自动推断**：当 PR 仅包含 `deployments/` 目录的变更时，可省略 `-profile` 和 `-stack` 参数，工作流会自动从变更文件推断。如果 PR 包含其他目录的变更，则必须手动指定这两个参数。

辅助脚本（`scripts/`）：

| 脚本 | 语言 | 依赖 | 作用 |
|------|------|------|------|
| `upload_iac_module.py` | Python 3.12 | `alibabacloud_iacservice20210806` | 调用自动化服务台的 `UploadModule` 接口上传代码包到模板并发布为模板版本 |
| `trigger_stack.py` | Python 3.12 | `alibabacloud_iacservice20210806` | 调用自动化服务台的 `TriggerStackExecution` 接口触发 Stack 执行 |
| `get_trigger_result.py` | Python 3.12 | `alibabacloud_iacservice20210806`, `PyYAML` | 轮询自动化服务台的 `GetStackExecutionResult` 接口获取结果。下载后解析 JSON 并格式化为 Markdown 表格。支持多 profile 并行轮询（多线程），默认超时 600 秒 |

---

# 前期准备

## 准备代码仓库

在 GitHub 中创建新仓库，将自动化服务台资源栈多账号管理脚手架项目中的 `ci-templates/direct-iacservice/github/` 目录下的所有文件复制到自己仓库的根目录中，提交并推送：

```bash
git add .
git commit -m "init"
git push --set-upstream origin main
```

---

## 初始化云端基础设施

### 1. 创建 RAM 用户 1：管理用户

用于开通自动化服务台资源的初始化配置，一次性操作。后续运维人员可使用当前用户登陆进行操作：

1. 登录 [RAM 控制台](https://ram.console.aliyun.com/)，创建用户并启用【OpenAPI 调用访问】
2. 为用户附加以下策略：
   - `AliyunRAMFullAccess`
   - 自定义权限策略 `IaCServiceStackFullAccess`（策略内容见 [docs/iam-policies.md](../../../docs/iam-policies.md)）

### 2. 创建 RAM 用户 2：流水线触发用户

用于 GitHub Actions 调用 IacService API，将 AccessKey 配置到 GitHub Secrets 中：

1. 创建用户并启用【OpenAPI 调用访问】
2. 附加自定义权限策略 `IaCServiceStackTriggerAccess`（策略内容见 [docs/iam-policies.md](../../../docs/iam-policies.md)）
3. 创建 AccessKey，记录 AccessKey ID 和 Secret，后续配置 GitHub Secrets 时会用到（本文以开发环境为例，假设密钥对命名为 `DEV_ACCESS_KEY_ID` / `DEV_ACCESS_KEY_SECRET`）

### 3. 创建 RAM 角色：自动化服务台执行角色

用于授权自动化服务台扮演该角色执行 Terraform 模板：

1. 在 RAM 控制台创建新的 RAM 角色
2. 配置信任策略，允许自动化服务台服务扮演此角色（信任策略内容见 [docs/iam-policies.md](../../../docs/iam-policies.md)）
3. 为该角色添加执行 Terraform 模板所需的权限策略（例如模板包含 ECS 实例，则需添加 ECS 相关权限）

> **多环境说明**：如有多个环境（dev、staging、prod 等），建议创建不同的 RAM Role，实现环境间完全隔离。

### 4. 创建自动化服务台模板

本步骤主要创建自动化服务台的模板。模板对应一份代码，每次代码修改后需要重新发布为新版本。同一份代码的不同变更对应不同的模板版本。

> **重要**：创建完成后请记录模板 ID（ModuleId），后续步骤需要用到。

**步骤 1：新建模板**

登录 [阿里云自动化服务台](https://iac.console.aliyun.com/)，点击【模板管理】，选择【新建模板】。

**步骤 2：选择模板来源**

选择【空白模版】方式。

**步骤 3：填写模板参数**

填写如下的参数后，点击【提交】：

| 参数 | 是否必填 | 说明 |
|------|----------|------|
| 模板名称 | 是 | 模板的名称，不可重复 |
| 模板描述 | 否 | 模板的描述信息 |
| 加入项目分组 | 否 | 可选择项目分组，用项目方式统一管理模板 |
| 标签 | 否 | 为模板添加的标签，便于分类管理 |

将创建后的 `moduleId` 填写到 `deployments/[env]/profile.yaml` 中的 `code_module_id` 对应的值：

```yaml
code_module_id: "mod-xxx"
```

> **多环境说明**：如有多个环境（dev、staging、prod 等），需要为每个环境执行独立的配置，使用不同的自动化服务台模板，实现环境间完全隔离。

---

## GitHub 配置

### 1. 创建 RAM 用户并配置自动化服务台权限

> **注意**：如果已在"初始化云端基础设施"步骤中创建了"流水线触发用户"，可直接使用该用户的 AccessKey，跳过本步骤。

创建一个只允许 API 访问的 RAM User，创建 AccessKey，授予以下最小权限：

```json
{
  "Version": "1",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "iacservice:UploadModule",
      "Resource": "acs:iacservice:*:*:module/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "iacservice:GetStackExecutionResult",
        "iacservice:TriggerStackExecution"
      ],
      "Resource": "acs:iacservice:*:*:stack/*"
    }
  ]
}
```

### 2. 配置 GitHub Secrets

将 AccessKey 添加到 GitHub 仓库的 Secrets，每个 Key 和 Value 分开存储，Key 名称对应 `profile.yaml` 中配置的密钥对名称：

| Secrets Key | Secrets Value |
|-------------|---------------|
| `DEV_ACCESS_KEY_ID` | `"xxx"` |
| `DEV_ACCESS_KEY_SECRET` | `"xxx"` |

> 如有多个环境，按环境前缀分别配置，例如 `STAGING_ACCESS_KEY_ID`、`STAGING_ACCESS_KEY_SECRET`、`PROD_ACCESS_KEY_ID`、`PROD_ACCESS_KEY_SECRET`

### 3. 声明工作流环境变量

在 `.github/workflows/pull-request-comment.yml` 和 `scheduled-check.yml` 的 `env` 块中声明 Secrets 引用：

```yaml
env:
  DEV_ACCESS_KEY_ID: ${{ secrets.DEV_ACCESS_KEY_ID }}
  DEV_ACCESS_KEY_SECRET: ${{ secrets.DEV_ACCESS_KEY_SECRET }}
  # 其他环境...
```

> **注意**：所有工作流文件（包括共享工作流）中如使用了硬编码的环境前缀，也需相应添加 Secrets 引用。

### 4. 初始化自动化服务台的模板

提交当前分支代码，手动触发 GitHub Actions 中的 **Scheduled Check** 工作流以初始化自动化服务台的模板（运行前选择当前分支，无需勾选 "Whether to run detect check"）。

> 如果运行报错，可能是 `deployments/` 下存在多个环境但未全部配置，删除多余环境目录或补全配置即可。

---

# GitHub Actions 工作流配置

仓库包含 5 个工作流文件，分为两类：

## 工作流文件

**主工作流**（由 GitHub 事件直接触发）：

| 工作流 | 触发方式 | 作用 |
|--------|---------|------|
| `pull-request-comment.yml` | PR 评论创建/编辑 | 解析 `iac terraform plan/apply` 命令，打包上传代码，触发自动化服务台执行，将结果回写 PR |
| `scheduled-check.yml` | 手动触发 / 定时任务 | 上传所有环境的源码包到自动化服务台的模板并发布为版本，可选触发漂移检测 |

**共享工作流**（被主工作流调用，可独立抽取复用）：

| 工作流 | 调用方 | 作用 |
|--------|-------|------|
| `shared-ci-get-pull-request-info.yml` | `pull-request-comment` | 校验 PR 可合并性，解析评论中的 `-profile`/`-stack` 参数，从变更文件自动推断受影响的 stacks |
| `shared-ci-upload-source-package.yml` | `scheduled-check` | 遍历所有 profile，执行 `make build-package` 构建源码包并上传自动化服务台的模板版本；单个 profile 失败不影响其他 |
| `shared-ci-upload-trigger-file.yml` | `pull-request-comment` | 为每个 profile 构建源码包、调用包含执行命令和变更 stacks 的自动化服务台接口的触发执行接口；任一失败立即终止 |

## 工作流依赖关系

```
pull-request-comment.yml
├── shared-ci-get-pull-request-info.yml    # Step 1: 解析命令、获取 PR 信息
├── shared-ci-upload-trigger-file.yml      # Step 2: 上传代码发布为对应的模板版本 + 触发执行
└── get_exec_result (内联 job)              # Step 3: 轮询结果 → 回写 PR Comment

scheduled-check.yml
├── shared-ci-upload-source-package.yml    # Step 1: 上传所有 profile 的源码包
└── get_exec_result (内联 job)              # Step 2: 轮询检测结果（仅 run_detect=true 时）
```

---

# 日常使用

## 创建自动化服务台资源栈

在进行代码的执行前，首先需要在阿里云自动化服务台上创建出对应的资源栈来承载代码的运行。在仓库代码初始化完成后，先通过 Scheduled Check 工作流将代码上传为自动化服务台的模板版本，再将对应的资源栈一次性创建出来。

到阿里云自动化服务台（https://iac.console.aliyun.com/stack）创建 Stack：

**步骤 1：创建资源栈**

点击【资源栈】，选择【创建资源栈】。

**步骤 2：填写资源栈信息**

| 参数 | 是否必填 | 说明 |
|------|----------|------|
| 资源栈名称 | 是 | 资源栈的名称，不可重复 |
| 描述 | 否 | 资源栈的描述信息 |
| 资源栈代码来源 | 是 | **Module**：选择前期准备中创建的自动化服务台模板<br>⚠️ **注意**：直连模式仅支持 Module 方式，不支持 OSS 方式 |
| 模板 ID/版本 | 是 | 选择前期准备中创建的模板 |
| 工作目录 | 是 | 资源栈配置文件在代码中的存放路径，如 `stacks/demo`                                                       |
| RAM 角色 | 是 | 选择前期准备中创建的角色，用于运行 Terraform 模板 |

**步骤 3：关联参数集**

选择或创建参数集，点击【下一步】。

> **保密参数说明**：参数集支持定义保密参数（如密码、API Token、访问密钥等），详情请参考 [参数集保密值使用指南](../../../docs/参数集敏感数据加密.md)。

**步骤 4：确认创建**

检查配置信息无误后，点击【创建】。

> **批量创建**：建议将 `stacks/` 目录下定义的所有 Stack 一次性全部创建。每个 Stack 对应一个独立的资源栈，创建完成后会自动执行首次 `plan`，可验证配置的正确性。

---

## 配置保密参数

参数集支持定义保密参数，用于安全存储密码、API Token、访问密钥等机密信息。保密值会自动基于阿里云 KMS 加密存储，并在控制台界面、运行日志、API 返回和 PR 回写结果中以掩码形式展示。

使用保密参数的基本流程：

1. **设置加密密钥**：进入**系统设置 > 加密设置**，选择服务密钥（系统自动创建）或用户主密钥（自行在 KMS 创建）。
2. **创建保密参数**：在参数集中新增参数时，勾选 **保密** 选项并保存。
3. **声明组件变量**：在 Stack Component 中，接收保密值的变量必须声明 `sensitive: true`。
4. **引用参数集**：在 Deployment 配置中通过 `store` 块引用参数集，格式为 `store.varset.<STORE_NAME>.<VARIABLE_NAME>`。

详细说明请参考 [参数集保密值使用指南](../../../docs/参数集敏感数据加密.md)。

---

## 变更流程

1. **创建开发分支，提交代码变更，推送到 GitHub**

```bash
git checkout -b dev
# 修改 stacks/ 或 deployments/[env]/ 中的配置...
git add .
git commit -m "update stack config"
git push --set-upstream origin dev
```

2. **创建 PR 到主分支（如 `main`），在 PR 评论中输入命令触发部署**

```bash
iac terraform plan -profile=dev -stack=demo
iac terraform apply -profile=dev -stack=demo
```

3. **执行结果会自动回写到 PR 评论，确认 apply 结果符合预期**

4. **评审通过，变更合入主干**

---

## 运行参数说明

通过 PR 评论传递运行参数，格式如下：

```
iac terraform plan/apply [-profile=<profileName>] [-stack=<StackName>]
```

**参数说明**：

| 参数 | 必填 | 说明 |
|------|------|------|
| `-profile` | 是* | 执行环境，对应 `deployments/` 目录下的环境名称 |
| `-stack` | 是* | 目标 stack 路径，支持目录嵌套，如 `demo/subDir` |

> *当 PR 仅包含 `deployments/` 目录的变更时，可省略这两个参数，工作流会自动从变更文件推断。

**注意事项**：

- 一个 PR 可重复使用直到 Merge 关闭，但建议每个任务创建独立 PR 便于追踪
- 每次 `apply` 前需先执行 `plan`，`plan` 可连续执行多次
- 不同环境的 `-profile` 对应不同的阿里云账号和资源，互不影响

---

# 运维参考

## 故障排查

| 问题 | 排查方法 |
|------|---------|
| 部署无响应 | 检查 GitHub Secrets 是否配置正确、IacService 服务是否正常、RAM 权限是否足够 |
| 命令无法识别 | 确认 PR Comment 格式为 `iac terraform plan/apply`，注意前缀必须完全匹配 |
| `make build-package` 失败 | 确认仓库根目录存在 Makefile 且包含 `build-package` 目标 |
| profile 上传失败 | 检查 `deployments/[env]/profile.yaml` 中的 `access_key_id`/`access_key_secret` 是否与 GitHub Secrets 中的 Key 名一致 |
| 代码上传失败 | 检查 `code_module_id` 是否正确、AK 是否有 IacService 相关权限 |
| 触发执行失败 | 检查 `code_module_version` 是否正确、资源栈是否已创建 |
| 结果获取超时 | 默认轮询超时 600 秒，检查 IacService 控制台确认 Stack 是否正在执行 |
| 多环境部分失败 | 查看 GitHub Actions 日志，`shared-ci-upload-source-package` 会列出失败的 profile 名称 |
| 查看详细错误信息 | 查看 GitHub Actions 运行日志，或检查 IacService 控制台中的执行记录 |
| 敏感参数解密失败 | 检查加密配置是否已初始化（系统设置 > 加密设置）、KMS 密钥是否处于启用状态且未被删除或禁用 |

---

## CI 依赖

GitHub Actions 运行时需要以下依赖：

| 依赖 | 用途 | 预置方式 |
|------|------|---------|
| **Python 3.12** | 运行 `upload_iac_module.py`、`trigger_stack.py` 和 `get_trigger_result.py` 脚本 | 使用 `actions/setup-python@v5` 自动安装 |
| **alibabacloud_iacservice20210806** | 阿里云 IacService SDK，用于上传代码包、触发执行和获取执行结果 | `pip install alibabacloud_iacservice20210806` |
| **PyYAML** | 解析 `profile.yaml` 和凭证文件 | `pip install PyYAML` |
| **make** | 执行 `make build-package` 构建源码包 | `ubuntu-latest` 镜像已预装 |
| **curl / jq** | 获取 PR 信息、解析 JSON 响应 | `ubuntu-latest` 镜像已预装 |

> **离线环境注意**：如 GitHub Actions 运行在自托管 Runner 且无法访问公网，需提前在 Runner 镜像中预装上述依赖，或将依赖包缓存到私有仓库/制品库中。

---

## 多环境配置

如需新增一个阿里云账号（例如新增 `test` 环境），需完成以下配置：

### 1. 创建 RAM Role 和自动化服务台模板

参考 [创建 RAM Role](../../../docs/iam-policies.md)、[创建自动化服务台模板](#3-创建自动化服务台模板) 章节，为新环境创建独立的 RAM Role 和模板。

### 2. 创建 deployments 配置

```bash
cp -r deployments/dev deployments/test
cd deployments/test
```

修改 `profile.yaml` 中的 `code_module_id` 为步骤 1 的 `moduleId`，新增 `profile.yaml` 中的 AK 变量名为新的 GitHub Secrets Key。

### 3. 配置 GitHub Secrets

在 GitHub 仓库 Settings → Secrets and variables → Actions 中添加：

| Secrets Key | Secrets Value |
|-------------|---------------|
| `TEST_ACCESS_KEY_ID` | 新账号的 AccessKey ID |
| `TEST_ACCESS_KEY_SECRET` | 新账号的 AccessKey Secret |

### 4. 声明 CI 环境变量

在 `.github/workflows/pull-request-comment.yml` 和 `scheduled-check.yml` 的 `env` 块中添加新账号的引用：

```yaml
env:
  # ... 其他环境
  TEST_ACCESS_KEY_ID: ${{ secrets.TEST_ACCESS_KEY_ID }}
  TEST_ACCESS_KEY_SECRET: ${{ secrets.TEST_ACCESS_KEY_SECRET }}
```

> **注意**：所有工作流文件（包括共享工作流）中如使用了硬编码的环境前缀（如 `DEV_`、`PROD_`），也需相应添加 `TEST_` 的引用。

### 5. 创建 Stack

到阿里云 IaC 控制台创建 Stack，选择步骤 1 中创建的 `moduleId` 和 RAM Role。参考 [创建自动化服务台资源栈](#创建自动化服务台资源栈) 章节。

### 6. 验证配置

提交配置后，手动触发 **Scheduled Check** 工作流，确认新环境的源码包上传成功。

> **账号隔离建议**：每个阿里云账号对应独立的 RAM Role、自动化服务台的模板资源，通过不同的 GitHub Secrets 管理凭证，实现完全隔离。推荐将不同环境划分到不同的阿里云主账号，通过阿里云资源目录（Resource Directory）进行统一治理。

---

# 附录

## 工作流设计详解

### pull-request-comment.yml

**触发条件**：PR 评论被创建或编辑时（`issue_comment: [created, edited]`）

**工作流程**：

1. **get_trigger_info** — 调用 `shared-ci-get-pull-request-info.yml`，解析评论中的命令、获取 PR 的 head SHA 和 base ref、推断受影响的 stacks
2. **process_trigger_file** — 调用 `shared-ci-upload-trigger-file.yml`，打包代码并上传到 IacService 模板，触发执行
3. **get_exec_result** — 轮询 IacService API 获取执行结果，格式化后回写到 PR 评论

**支持的命令格式**：

```
iac terraform plan [-profile=<profile>] [-stack=<stack>]
iac terraform apply [-profile=<profile>] [-stack=<stack>]
```

> 当 PR 仅包含 `deployments/` 目录的变更时，工作流会自动从变更文件推断 profiles 和 stacks，此时 `-profile` 和 `-stack` 参数可选。如果 PR 包含其他目录的变更，则必须手动指定这两个参数。

### scheduled-check.yml

**触发条件**：手动触发（`workflow_dispatch`）或定时自动运行

**工作流程**：

1. **process_code** — 调用 `shared-ci-upload-source-package.yml`，处理所有 profiles（`profiles: all`）的源码包上传到 IacService 模板；若启用 `run_detect` 则额外触发检测
2. **get_exec_result** — 仅在 `run_detect=true` 或定时触发时执行，获取并输出执行结果

**主要用途**：

- **偏差检测（Drift Detection）**：通过 `run_detect=true` 触发，自动检测云上实际资源与 Terraform 配置的偏差
- **源码包初始化**：首次配置时上传源码包到 IacService 模板，为 PR 部署做准备
- **代码完整性验证**：定期验证所有环境的代码可正常构建

> **代码来源**：默认上传 `main` 分支的代码。偏差检测会对比云上资源状态与 `main` 分支配置，发现偏差后生成检测报告。

### 共享工作流

**shared-ci-get-pull-request-info.yml**

获取 PR 详细信息并解析命令参数。主要功能：验证 PR 可合并性、提取 head SHA 和 base ref、解析 `-profile`/`-stack` 参数、从变更文件自动推断受影响的 stacks。

主要输出：`command`、`base_ref`、`head_sha`、`all_changed_stacks`

**shared-ci-upload-source-package.yml**

构建源码包并上传到 IacService 模板，支持多 profile 批量处理。对每个 profile 执行 `make build-package PROFILE=<profile>` 构建代码包，通过 `upload_iac_module.py` 上传到 IacService 模板。单个 profile 失败不影响其他 profile，全部处理完成后统一汇总失败信息。

**shared-ci-upload-trigger-file.yml**

上传源码包到 IacService 模板并触发执行接口，启动自动化服务台的执行。解析 `all_changed_stacks` 参数，为每个 profile 执行以下操作：

1. 构建源码包
2. 调用 `upload_iac_module.py` 上传到 IacService 模板，获取 `version_id`
3. 调用 `trigger_stack.py` 触发 Plan 或 Apply 执行，获取 `trigger_id`

采用快速失败策略，任一 profile 失败时立即终止。

触发信息示例：

```
Profile: dev, Version ID: v123456
Profile: dev, Trigger ID: trg-abc123...
```
