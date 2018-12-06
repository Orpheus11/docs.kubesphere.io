---
title: "示例六 - Jenkinsfile out of SCM" 
---

示例一通过代码仓库中的 Jenkinsfile 构建流水线，需要对声明式的 Jenkinsfile 有一定的基础。而 Jenkinsfile out of SCM 不同于 [Jenkinsfile in SCM](../jenkinsfile-in-scm)，其代码仓库中可以无需 Jenkinsfile，支持用户在控制台通过可视化的方式构建流水线或编辑 Jenkinsfile 生成流水线，用户操作界面更友好。

本示例演示基于 [示例一 - Jenkinsfile in SCM](../jenkinsfile-in-scm)，通过可视化构建流水线 (包含示例一的前六个阶段)，最终将一个文档网站部署到 KubeSphere 集群中的开发环境且能够通过公网访问。若熟悉了示例一的流程后，对于示例二的手动构建步骤就很好理解了。为方便演示，本示例仍然以 GitHub 代码仓库 [devops-docs-sample](https://github.com/kubesphere/devops-docs-sample) 为例，可预先将其 Fork 至您的代码仓库中。

构建可视化流水线共包含以下 6 个阶段 (stage)，先通过一个流程图简单说明一下整个 pipeline 的工作流：

![流程图](/cicd-pipeline-02.svg)

> 详细说明每个阶段所执行的任务：
> - **阶段一. Checkout SCM**: 拉取 GitHub 仓库代码
> - **阶段二. Get dependencies**: 通过包管理器 [yarn](https://yarnpkg.com/zh-Hans/) 安装项目的所有依赖
> - **阶段三. Unit test**: 单元测试，如果测试通过了才继续下面的任务
> - **阶段四. Build**:  执行项目中 `package.json` 的 scripts 里的 build 命令，即生成静态网站的命令。
> - **阶段五. Build & push snapshot image**: 构建镜像，并将 tag 为 `SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER` 推送至 DockerHub (其中 `$BUILD_NUMBER` 为 pipeline 活动列表的运行序号)。
> - **阶段六. Deploy to dev**: 将 master 分支部署到 Dev 环境，此阶段需要审核。

## 前提条件

- 已有 [DockerHub](http://www.dockerhub.com/) 的账号。
- 本示例的代码仓库以 GitHub 为例，确保已有 [GitHub](https://github.com/) 账号，且已 Fork 了示例代码仓库。
- 已创建了 DevOps 工程，若还未创建请参考 [创建 DevOps 工程](../../devops/devops-project/#创建-devops-工程)。

## 演示视频

<video controls="controls" style="width: 100% !important; height: auto !important;">
  <source type="video/mp4" src="https://kubesphere-docs.pek3b.qingstor.com/video/jenkinsfile-out-of-scm.mp4">
</video>

## 创建凭证

本示例创建流水线时需要 DockerHub 和 Kubernetes (创建 KubeConfig 用于接入正在运行的 Kubernetes 集群) 的 2 个凭证 (credentials) ，先依次创建这 2 个凭证。

> 注意：
> - 若示例二与示例一在同一 DevOps 工程，且示例一已创建了这两个凭证，则无需再次创建，因为同一 DevOps 工程下的凭证对流水线是共用的。
> - 若代码仓库属于私有仓库如 Gitlab，可能还需要创建这类账户的凭证。

### 第一步：创建 DockerHub 凭证

1、在左侧的工程管理菜单下。点击 `凭证管理`，进入凭证管理界面，界面会展示当前工程所需的所有可用凭证。

![credential_page](/devops_credentials.png)

2、点击创建按钮，创建一个用于 DockerHub 登录的凭证。

- 凭证 ID：必填，此 ID 将用于仓库中的 Jenkinsfile，可自定义，此处命名为 **dockerhub-id**。
- 类型：选择 **账户凭证**。
- 用户名/密码：输入您个人的 DockerHub 用户名和密码。
- 描述信息：介绍凭证，比如此处可以备注为 DockerHub 登录凭证。

![Dockerhub 凭证](/dockerhub-credential.png)

### 第二步：创建 kubeconfig 凭证

创建一个类型为 **kubeconfig** 的凭证，凭证 ID 命名为 **demo-kubeconfig**，用于访问接入正在运行的 Kubernetes 集群，在流水线部署步骤将用到该凭证。注意，此处的 Content 将自动获取当前 KubeSphere 中的 kubeconfig 文件内容，若部署至当前 KubeSphere 中则无需修改，若部署至其它 Kubernetes 集群，则需要将其 kubeconfig 文件 (一般在 `$HOME/.kube/config`) 的内容粘贴至 Content 中。

至此，2 个凭证已经创建完成。

![凭证列表](/credentials-in-sample02.png)

## 创建流水线

参考以下步骤，创建并运行一个完整的流水线。

### 第一步：填写基本信息

1、进入已创建的 DevOps 工程，选择左侧 **流水线** 菜单项，然后点击 **创建流水线**。

![创建流水线](/create_manual_pipeline.png)

2、在弹出的窗口中，输入流水线的基本信息。
- 名称：为流水线起一个简洁明了的名称，便于理解和搜索。
- 描述信息：简单介绍流水线的主要特性，帮助进一步了解流水线的作用。
- 代码仓库：此处不选择代码仓库

![基本信息](/manaul_pipeline_info.png)

### 第二步：高级设置

1、填写基本信息后，进入高级设置页面。高级设置支持对流水线的构建记录、参数化构建、定期扫描等设置的定制化。例如 **丢弃旧的构建** 可以决定何时应丢弃项目的构建记录。构建记录包括控制台输出，存档工件以及与特定构建相关的其他元数据。保持较少的构建可以节省 Jenkins 所使用的磁盘空间。

因此，此处需勾选 `丢弃旧的构建`，本示例中 **保留构建的天数** 设置为 `1`，**保持构建的最大个数** 设置为 `3` (这个数值大小可根据团队习惯来设置)。

![丢弃旧的构建](/pipeline-advanced-setting-1.png)

2、参数化构建过程允许您在进行构建时传入一个或多个参数，此处定义的每个参数应具有唯一的名称。当参数化项目时，构建会被替换为参数化构建，将提示用户为每个定义的参数输入值。如果不输入任何内容，构建将以每个参数的默认值进行。如果项目的构建是自动启动，例如，由定时触发器启动，这时将使用参数的默认值进行触发。本示例在参数化构建中添加 **字符串参数 (String)** 为例，演示如何在流水线中使用该参数。

如下添加两个字符串参数，将在流水线的 docker 命令中使用该参数。完成后点击 **创建**，创建完成后页面将自动跳转至流水线的可视化编辑页面，

|名称|默认值|描述信息|
|---|---|---|
|DOCKERHUB_ORG|您的 DockerHub 账号|DockerHub 账号|
|APP_NAME|devops-docs-sample|应用名称|

![高级设置](/pipeline-advanced-setting.png)

3、设置 **构建触发器**，此处勾选 **定时构建**，日程表 (cron) 填写 `H H * * *`，表示每天构建一次 (不限具体时刻)。这里的定时构建是提供类似 Linux cron 的功能来定期执行此流水线，定时构建的语法以下作一个简单释义，语法详见 [Jenkins 官方文档](https://jenkins.io/doc/book/pipeline/syntax/#cron-syntax)。

> 定时构建语法：
> `* * * * *`
> - 第一个 * : 表示分钟，取值 0~59
> - 第二个 * : 表示小时，取值 0~23
> - 第三个 * : 表示一个月的第几天，取值 1~31
> - 第四个 * : 表示第几月，取值 1~12
> - 第五个 * : 表示一周中的第几天，取值 0~7，其中 0 和 7 代表的都是周日
>
> 常用的定时构建比如：
> - `H H/2 * * *`: 每两小时构建一次
> - `0 18 * * *`: 每天 18:00 下班前定时构建一次
> - `H H(9-18)/2 * * 1-5`: 在工作日的 9 AM ~ 6 PM 期间每两个小时构建一次 (或许在 10:38 AM, 12:38 PM, 2:38 PM, 4:38 PM)

## 可视化编辑流水线

可视化流水线共包含 6 个阶段 (stage)，以下依次说明每个阶段中分别执行了哪些步骤和任务。

### 阶段一：Checkout SCM

1、可视化编辑页面，分为结构编辑区域和内容编辑区域。通过构建流水线的每个阶段 (stage) 和步骤 (step) 即可自动生成 Jenkinsfile，用户无需学习 Jenkinsfile 的语法，非常方便。当然，平台也支持手动编辑 Jenkinsfile 的方式，流水线分为 “声明式流水线” 和 “脚本化流水线”，可视化编辑支持声明式流水线。Pipeline 语法参见 [Jenkins 官方文档](https://jenkins.io/doc/book/pipeline/syntax/)。

如下，代理 (Agent) 的类型选择 `node`，label 填写 nodejs。

![代理设置](/pipeline_agent.png)

2、点击左侧结构编辑区域的 **“+”** 号，增加一个阶段 (Stage)，命名为 **checkout SCM**，然后在此阶段下点击 `添加步骤`。右侧选择 `git`，此阶段通过 Git 拉取仓库的代码，弹窗中填写的信息如下：

 - Url: GitHub 仓库的 url `https://github.com/kubesphere/devops-docs-sample.git`。
 - 凭证 ID: 无需填写，若是私有仓库 如 Gitlab 则预先创建并填写其凭证 ID。
 - 分支：不填则默认为 master 分支，此处无需填写。
   
![拉取源代码](/checkout-scm.png)

### 阶段二：Get dependencies

1、在第一个阶段右侧点击  **“+”**  继续增加一个阶段，此阶段用于获取依赖，命名为 **get dependencies**。由于这个阶段需要在容器中执行 shell 脚本，因此点击 **添加步骤**，在右侧内容编辑区域首先选择 `container` 并命名为 **nodejs**。

![获取依赖](/get-dependencies.png)

2、然后在右侧内容编辑区域点击 **添加嵌套步骤**，选择 `shell`，shell 命令填写 `yarn`，`yarn` 等同于 `yarn install`。本文档网站使用包管理器 [yarn](https://yarnpkg.com/zh-Hans/) 管理依赖，执行该命令安装项目的所有依赖。

![添加嵌套步骤](/pipeline-shell.png)

### 阶段三：Unit test

同上，新添加一个阶段用于在容器中执行单元测试，名称为 `unit test`。点击 **添加步骤** 选择 `container`，命名为 `nodejs`；然后点击 **添加嵌套步骤**，选择 `shell`，shell 命令填写 `yarn test`。如果测试通过了才允许继续下面的任务。

![添加单元测试](/yarn-test.png)

### 阶段四：Build

同上，新添加一个阶段 `build` 用于在容器中执行生成静态网站的命令，名称为 `build`。点击 **添加步骤** 选择 `container`，命名为 `nodejs`；然后点击 **添加嵌套步骤**，选择 `shell`，shell 命令填写 `yarn build`，用于执行项目中 `package.json` 的 scripts 里的 build 命令。如果测试通过了才允许继续下面的任务。

![创建发布包](/pipeline-build.png)

### 阶段五：Build and push snapshot image
1、新添加一个阶段，该阶段用于在容器中构建 snapshot 镜像，并将 tag 为 `SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER` 推送至 DockerHub (其中 `$BRANCH_NAME` 为分支名称，`$BUILD_NUMBER` 为 pipeline 活动列表的运行序号)。

2、点击 **添加步骤** 选择 `container`，名称为 `nodejs`；然后点击 **添加嵌套步骤**，选择 `shell`，shell 命令填写如下一行 docker 命令：

```shell
docker build -t docker.io/$DOCKERHUB_ORG/$APP_NAME:SNAPSHOT-$BUILD_NUMBER .

```

![构建镜像](/docker-build-image.png)

3、考虑用户信息安全，账号类信息都不以明文出现在脚本中，而以变量的方式。点击 **添加步骤** 在右侧选择 `withCredentials`，并填写如下信息用于登录 DockerHub。

- 凭证 ID：选择之前创建的 DockerHub 凭证
- 密码变量：`DOCKER_PASSWORD`
- 用户名变量：`DOCKER_USERNAME`

![添加凭证信息](/withcredentials-info.png)

4、注意，下一步需要在 **withCredentials** 中依次添加两个 **嵌套步骤**，并且都选择 `shell`，用于登录 DockerHub 并推送镜像。两条 shell 命令填写依次如下：

```bash
echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin

docker push docker.io/$DOCKERHUB_ORG/$APP_NAME:SNAPSHOT-$BUILD_NUMBER
```
![推送 snapshot](/build-push-snapshot.png)

### 阶段六：Deploy to dev

1、最后一步，点击  **“+”**  继续增加一个阶段，此阶段用于人工审核，并部署到 Dev 环境，命名为 **deploy to dev**。因为在实际项目过程中，比如开发者提交了一次代码，测试也通过了，镜像也打包上传了，但是这个版本并不一定就是要立刻上线到生产环境的，可能需要将该版本先发布到开发环境、测试环境、或者预览环境之类，所以通常需要在 CD 的环节增加人工审核。

2、点击 **添加步骤**，右侧选择 `input`，该步骤用于人工审核，当流水线执行至审核步骤时将暂停，待审核通过后才可以继续进行，审核信息如下：

- id：可选，填写步骤的名称
- 消息 (message)：必填，在用户人工审核时呈现给用户，此处可填 `Deploy to dev?`
- 审核者 (submitter)：此处为演示方便暂使用当前用户审核，审核者为空则默认此工程下包括当前用户在内的所有用户 (不限角色) 都具有审核权限。注意，在正式环境中可输入 **用户名** 来指定用户来人工审核。

![审核流水线](/deploy-to-dev-info.png)

3、在当前阶段点击 **添加步骤**，右侧选择 `kubernetesDeploy`，这是 [Kubernetes Continuous Deploy Plugin](https://jenkins.io/doc/pipeline/steps/kubernetes-cd/#kubernetes-continuous-deploy-plugin) 中的函数，该步骤可将项目中指定路径的 yaml 模板部署到 Kubernetes 中，配置信息如下：

- Kubeconfig：选择之前创建的 kubeconfig 凭证
- 配置文件路径 (configs)：yaml 模板在项目中的相对路径，填写 `deploy/no-branch-dev/**`
- 在配置中启用变量替换 (enableConfigSubstitution)：勾选，允许在流水线中通过变量动态传值 (比如以上用到的 `$DOCKER_PASSWORD`、`$APP_NAME` )

![部署到 Kubernetes 配置](/Kubernetes-deploy-info.png)

至此，流水线的 6 个阶段都已构建完成，点击 **保存**，可视化创建的 pipeline 会默认生成为 Jenkinsfile 文件。

![流水线总览](/pipeline-overview.png)

## 运行流水线

手动构建的流水线在平台中需要手动运行，点击 **运行**，输入参数弹窗中可编辑之前定义的两个字符串参数，此处无需修改，点击确定，流水线将开始运行。

![运行流水线](/run-pipeline.png)

在 **活动** 列表中可以看到流水线的运行状态，点击活动项可查看其运行活动的具体情况，例如以下查看 **运行序号** 为 `2` 的活动。

> 说明：流水线刚启动时可能仅显示其日志输出而无法看到其图形化运行的页面，这是因为它有个初始化的过程，流水线刚开始运行时，slave 启动并开始解析和执行流水线自动生成的 jenkinsfile，待初始化完成即可看到图形化流水线运行的页面。

![](/pipeline-status.png)

在活动运行序号 `2` 中，由于我们在最后一个阶段添加了 `input` 即人工审核步骤，因此流水线运行至此将停止等待人工手动触发，此时审核者可以测试构建的镜像并进一步审核整个流程，若审核通过则点击 **继续**，最终将部署到 Kubernetes 的 dev 环境中。

![审核部署阶段](/pipeline-deploy-to-dev.png)

## 查看流水线
   
1、点击流水线中 `活动` 列表下当前正在运行的流水线序列号，页面展现了流水线中每一步骤的运行状态。黑色框标注了流水线的步骤名称，示例中流水线的 6 个 stage 就是以上创建的六个阶段。
   
![run_status](/pipeline_status.png)

2、当前页面中点击右上方的 `查看日志`，查看流水线运行日志。页面展示了每一步的具体日志、运行状态及时间等信息，点击左侧某个具体的阶段可展开查看其具体的日志，若出现错误可根据日志信息来分析定位问题，日志支持下载至本地。
   
![log](/pipeline_log.png)

## 验证运行结果

若流水线的每一步都能执行成功，那么流水线最终 build 的 Docker 镜像也将被成功地 push 到 DockerHub 中，我们在 Jenkinsfile 中已经配置过 Docker 镜像仓库，登录 DockerHub 查看镜像的 push 结果，可以看到 tag 为 snapshot、TAG_NAME(v0.0.1)、latest 的镜像已经被 push 到 DockerHub，并且在 GitHub 中也生成了一个新的 tag，最终以 deployment 和 service 部署到 KubeSphere 的 dev 和 production 环境中。若需要在外网访问，可能需要进行端口转发并开放防火墙，即可访问成功部署的文档网站示例的首页。

|环境|访问地址| 所在项目 (Namespace) | 部署 (Deployment) |
|---|---|---|---|
|Dev| 公网IP (EIP) + 30880| kubesphere-system-dev| ks-docs-sample-dev|

查看推送到 DockerHub 的镜像，可以看到 `devops-docs-sample` 就是 **APP_NAME** 的值，而 **Tag Name** 则是 `SNAPSHOT-$BUILD_NUMBER` 的值 (`$BUILD_NUMBER` 对应活动的运行序号 **2**)。
  
![查看镜像](/deveops-dockerhub.png)

访问部署到 KubeSphere 的 Dev 环境的服务。

**Dev 环境**
![](/docs-home-dev-preview.png)


至此，创建一个 Jenkinsfile Out of SCM 类型的流水线已经完成了，若创建过程中遇到问题，可参考 [常见问题](../../faq)。
