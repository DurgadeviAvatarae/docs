---
title: 使用构件持久化工作流程数据
intro: 构件允许您在工作流程完成后，分享工作流程中作业之间的数据并存储数据。
product: '{% data reusables.gated-features.actions %}'
redirect_from:
  - /articles/persisting-workflow-data-using-artifacts
  - /github/automating-your-workflow-with-github-actions/persisting-workflow-data-using-artifacts
  - /actions/automating-your-workflow-with-github-actions/persisting-workflow-data-using-artifacts
versions:
  free-pro-team: '*'
  enterprise-server: '>=2.22'
---

{% data reusables.actions.enterprise-beta %}
{% data reusables.actions.enterprise-github-hosted-runners %}

### 关于工作流程构件

构件允许您在作业完成后保留数据，并与同一工作流程中的另一个作业共享该数据。 构件是指在工作流程运行过程中产生的文件或文件集。 例如，在工作流程运行结束后，您可以使用构件保存您的构建和测试输出。 对于推送和拉取请求， {% data variables.product.product_name %} 会将构件存储 90 天。 每当有人推送新提交到拉取请求时，拉取请求的保存期就会重新开始计算。

以下是您可以上传的一些常见构件：

- 日志文件和核心转储文件
- 测试结果、失败和屏幕截图
- 二进制或压缩文件
- 压力测试性能输出和代码覆盖结果

{% if currentVersion == "free-pro-team@latest" %}

存储构件时使用存储空间 {% data variables.product.product_name %}。 {% data reusables.github-actions.actions-billing %} 更多信息请参阅“[管理 {% data variables.product.prodname_actions %} 的计费](/github/setting-up-and-managing-billing-and-payments-on-github/managing-billing-for-github-actions)”。

{% else %}

项目会占用外部 Blob 存储上的存储空间，该存储为 {% data variables.product.prodname_actions %} 配置 {% data variables.product.product_location %}。

{% endif %}

构件会在工作流程运行过程中上传，您可以在 UI 中查看构件的名称和大小。 当构件使用 {% data variables.product.product_name %} UI 下载时， 作为构件一部分单独上传的所有文件都会压缩到一个 zip 文件中。 这意味着计费是根据上传的构件大小而不是 zip 文件的大小计算的。

{% data variables.product.product_name %} 提供两项可用于上传和下载构建构件的操作。 更多信息请参阅 [action/upload-artifact](https://github.com/actions/upload-artifact) 和 [download-artifact](https://github.com/actions/download-artifact) 操作。

要在作业之间共享数据：

* **上传文件**：为上传的文件提供名称并在作业结束前上传数据。
* **下载文件**：您只能下载在同一工作流程运行过程中上传的构件。 下载文件时，您可以通过名称引用该文件。

作业步骤共享运行器机器的相同环境，但在其各自的进程中运行。 要在作业的步骤之间传递数据，您可以使用输入和输出。 有关输入和输出的更多信息，请参阅“[{% data variables.product.prodname_actions %} 的元数据语法](/articles/metadata-syntax-for-github-actions)”。

### 在工作流程中作业之间传递数据

您可以使用 `upload-artifact` 和 `download-artifact` 操作在工作流程中的作业之间共享数据。 此示例工作流程说明如何在相同工作流程中的任务之间传递数据。 更多信息请参阅 [action/upload-artifact](https://github.com/actions/upload-artifact) 和 [download-artifact](https://github.com/actions/download-artifact) 操作。

依赖于以前作业构件的作业必须等待依赖项成功完成。 此工作流程使用 `needs` 关键词确保 `job_1`、 `job_2` 和 `job_3` 按顺序运行。 例如， `job_2` 需要 `job_1` 使用 `needs: job_1` 语法。

作业1执行以下步骤：
- 执行数学计算并将结果保存到名为 `math-home-work.txt` 的文本文件。
- 使用 `upload-artifact` 操作上传名为 `homework` 的 `math-homework.txt` 文件。 该操作将文件置于一个名为 `homework` 的目录中。

作业 2 使用上一个作业的结果：
- 下载上一个作业中上传的 `homework` 构件。 默认情况下， `download-artifact` 操作会将构件下载到该步骤执行的工作区目录中。 您可以使用 `path` 输入参数指定不同的下载目录。
- 读取 `homework/math-homework.txt` 文件中的值，进行数学计算，并将结果保存到 `math-homework.txt`。
- 更新 `math-homework.txt` 文件。 此上传会覆盖之前的上传，因为两次上传共用同一名称。

作业 3 显示上一个作业中上传的结果：
- 下载 `homework` 构件。
- 将数学方程式的结果打印到日志中。

此工作流程示例中执行的完整数学运算为 `(3 + 7) x 9 = 90`。

```yaml
name: Share data between jobs

on: [push]

jobs:
  job_1:
    name: Add 3 and 7
    runs-on: ubuntu-latest
    steps:
      - shell: bash
        run: |
          expr 3 + 7 > math-homework.txt
      - name: Upload math result for job 1
        uses: actions/upload-artifact@v2
        with:
          name: homework
          path: math-homework.txt

  job_2:
    name: Multiply by 9
    needs: job_1
    runs-on: windows-latest
    steps:
      - name: Download math result for job 1
        uses: actions/download-artifact@v2
        with:
          name: homework
      - shell: bash
        run: |
          value=`cat math-homework.txt`
          expr $value \* 9 > math-homework.txt
      - name: Upload math result for job 2
        uses: actions/upload-artifact@v2
        with:
          name: homework
          path: math-homework.txt

  job_3:
    name: Display results
    needs: job_2
    runs-on: macOS-latest
    steps:
      - name: Download math result for job 2
        uses: actions/download-artifact@v2
        with:
          name: homework
      - name: Print the final result
        shell: bash
        run: |
          value=`cat math-homework.txt`
          echo The result is $value
```

![要在作业之间传递数据以执行数学工作流程](/assets/images/help/repository/passing-data-between-jobs-in-a-workflow.png)

### 在工作流程运行之间共享数据

一个工作流程结束后，您可以通过在 **Actions（操作）**选项卡中找到工作流程运行，下载 {% data variables.product.product_name %} 上已上传构件的压缩文件。 您还可以使用 {% data variables.product.prodname_dotcom %} REST API 来下载构件。 更多信息请参阅“[构件](/v3/actions/artifacts/)”。

如果需要从以前的工作流运行访问项目，可以使用 REST API {% data variables.product.product_name %} 检索工件。 有关详细信息，请参阅"获取[项](/rest/reference/actions#artifacts)。

### 上传构建和测试构件

您可以创建持续集成 (CI) 工作流程来构建和测试您的代码。 关于使用 {% data variables.product.prodname_actions %} 执行 CI 的更多信息，请参阅“[关于持续集成](/articles/about-continuous-integration)”。

构建和测试代码的输出通常会生成可用于调试测试失败的文件和可部署的生产代码。 您可以配置一个工作流程来构建和测试推送到仓库中的代码，并报告成功或失败状态。 您可以上传构建和测试输出，以用于部署、调试失败的测试或崩溃以及查看测试套件范围。

您可以使用 `upload-artifact` 操作上传构件。 上传构件时，您可以指定单个文件或目录，或多个文件或目录。 您还可以排除某些文件或目录，以及使用通配符模式。 我们建议您为构件提供名称，但如果未提供名称，则会使用 `artifact` 作为默认名称。 有关语法的信息，请参阅 {% if currentVersion == "free-pro-team@latest" %}[操作/上传](https://github.com/actions/upload-artifact) 执行{% else %} `/上传项目` 操作 {% data variables.product.product_location %}{% endif %}。

#### 示例

例如，您的仓库或 Web 应用程序可能包含必须转换为 CSS 和 JavaScript 的 SASS 和 TypeScript 文件。 假设您的构建配置输出 `dist` 目录中已编译的文件，如果所有测试均已成功完成，则可将 `dist` 目录中的文件部署到您的 Web 应用程序服务器。

```
|-- hello-world (repository)
|   └── dist
|   └── tests
|   └── src
|       └── sass/app.scss
|       └── app.ts
|   └── output
|       └── test
|   
```

本例向您展示如何创建 Node.js 项目的工作流程，该项目在 `src` 目录中 `builds` 代码，并在 `tests` 目录中运行测试。 您可以假设运行 `npm test` 会产生名为 `code-coverage.html`、存储在 `output/test/` 目录中的代码覆盖报告。

工作流程上传 `dist` 目录中的生产构件，但不包括任何 markdown 文件。 它还会上传 `code-coverage.html` 报告作为另一个构件。

```yaml
name: Node CI

on: [push]

jobs:
  build_and_test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v2
      - name: npm install, build, and test
        run: |
          npm install
          npm run build --if-present
          npm test
      - name: Archive production artifacts
        uses: actions/upload-artifact@v2
        with:
          name: dist-without-markdown
          path: |
            dist
            !dist/**/*.md
      - name: Archive code coverage results
        uses: actions/upload-artifact@v2
        with:
          name: code-coverage-report
          path: output/test/code-coverage.html
```

![工作流程上传构件工作流程运行的图像](/assets/images/help/repository/upload-build-test-artifact.png)

### 下载或删除构件

在工作流程运行期间，您可以下载以前在同一工作流程运行中上传的构件。 在工作流程运行完成后，您可以使用工作流程运行历史记录在 GitHub 上下载或删除构件。

#### 在工作流程运行期间下载构件

[actions/download-artifact](https://github.com/actions/download-artifact) 操作可用于下载工作流程运行期间之前上传的构件。

{% note %}

**注**：您只能下载在同一工作流程运行过程中上传的构件。

{% endnote %}

指定构件的名称以下载单个构件。 如果在未指定名称的情况下上传构件目，则使用默认名称 `artifact`。

```yaml
- name: Download a single artifact
  uses: actions/download-artifact@v2
  with:
    name: my-artifact
```

您还可以不指定名称而下载工作流程运行中的所有构件。 如果您在处理大量构件，此功能非常有用。

```yaml
- name: Download all workflow run artifacts
  uses: actions/download-artifact@v2
```

如果下载所有工作流程运行的构件，则每个构件使用其名称目创建目录。

For more information on syntax, see the [actions/download-artifact](https://github.com/actions/download-artifact) action.

#### 工作流程运行完成后下载和删除构件

构件在 90 天后自动过期，但您始终可以在构件于 {% data variables.product.product_name %} 上过期之前删除它们，回收已经使用的 {% data variables.product.prodname_actions %} 存储。

{% warning %}

**警告：** 构件一旦删除，便无法恢复。

{% endwarning %}

{% data reusables.repositories.navigate-to-repo %}
{% data reusables.repositories.actions-tab %}
{% data reusables.repositories.navigate-to-workflow %}
{% data reusables.repositories.view-run %}
1. 要下载构件，请使用 **Artifacts（构件）**下拉菜单，然后选择要下载的构件。 ![下载构件下拉菜单](/assets/images/help/repository/artifact-drop-down.png)
1. 要删除构件，请使用 **Artifacts（构件）**下拉菜单，然后单击 {% octicon "trashcan" aria-label="The trashcan icon" %}。 ![删除构件下拉菜单](/assets/images/help/repository/actions-delete-artifact.png)

{% if currentVersion == "free-pro-team@latest" %}

### 延伸阅读

- "[管理 {% data variables.product.prodname_actions %} 的计费](/github/setting-up-and-managing-billing-and-payments-on-github/managing-billing-for-github-actions)".

{% endif %}
