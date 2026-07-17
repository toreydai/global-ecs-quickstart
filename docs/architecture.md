# 架构文档

本仓库包含 10 个基础主线 Demo + 4 个进阶专题 Demo，这里不做全量架构图汇总，只对其中组件交互较复杂、值得可视化的 Demo 提供架构图；其余 Demo 请直接看对应的 `docs/demoXX-*.md`。

选取标准：跨多个 AWS 服务编排、存在明确的多步骤调用链路。据此选出 3 个 Demo：**Demo10（CodePipeline CI/CD）**、**Demo12（Service Discovery / Cloud Map）**、**Demo13（CodeDeploy 蓝绿发布）**。

---

## Demo10 — CodePipeline 自动部署 ECS

Demo10 搭建一条完整的 CI/CD 流水线：源码提交到 CodeCommit 触发 Pipeline，Source 阶段拉取代码，Build 阶段用 CodeBuild 执行 `buildspec.yml`（登录 ECR → docker build/tag/push → 生成 `imagedefinitions.json`），Deploy 阶段用 CodePipeline 原生的 ECS 部署 Action 读取 `imagedefinitions.json` 并滚动更新 `demo-ecs-web` Service。三个 IAM Role（CodeBuild Role、CodePipeline Role，以及 Pipeline 对 Task/Execution Role 的 `iam:PassRole`）串联起整条链路。

```mermaid
flowchart LR
  Dev["开发者/操作机"]

  subgraph AWS["AWS us-east-1"]
    CC["CodeCommit\ndemo-ecs-app"]
    S3["S3 Artifact Bucket"]

    subgraph Pipeline["CodePipeline: demo-ecs-pipeline"]
      Source["Source Stage"]
      Build["Build Stage"]
      Deploy["Deploy Stage\n(Action: ECS)"]
    end

    CB["CodeBuild Project\ndemo-ecs-build\n(buildspec.yml)"]
    ECR["ECR 私有仓库"]
    ECSSvc["ECS Service: demo-ecs-web\n(Fargate)"]
    ALB["ALB :8080"]
  end

  Dev -->|"git push"| CC
  CC -->|"触发"| Source
  Source -->|"SourceOutput 制品"| S3
  Source --> Build
  Build -->|"StartBuild"| CB
  CB -->|"docker login/build/push"| ECR
  CB -->|"生成 imagedefinitions.json"| S3
  Build --> Deploy
  Deploy -->|"读取 imagedefinitions.json\nUpdateService"| ECSSvc
  ECSSvc --> ALB
```

流水线的核心是三次角色切换：CodeBuild Role 只能推镜像和写 Artifact Bucket；CodePipeline Role 除了编排各阶段，还需要对 Execution Role / Task Role 有 `iam:PassRole` 权限才能把新 Task Definition 交给 ECS 部署——这是本 Demo 里最容易被忽略、也最值得在架构图之外单独理解的一处权限细节。

---

## Demo12 — Service Discovery 与 Service Connect

Demo12 用 Cloud Map 私有 DNS Namespace（`demo.local`）实现 ECS 服务间的服务发现：Backend Service 创建时把自己注册进 Cloud Map（`backend.demo.local`），Frontend Task 通过 DNS 解析该域名访问 Backend；Demo 还刻意演练了 Backend 缩容到 0 后 Cloud Map 实例清空、Frontend 访问失败的故障场景，再恢复验证。

```mermaid
flowchart TB
  subgraph AWS["AWS us-east-1"]
    CloudMap["Cloud Map\nPrivate DNS Namespace: demo.local"]

    subgraph Cluster["ECS Cluster: demo-ecs (Fargate)"]
      Backend["Backend Service\n(demo-ecs-backend, desiredCount=2)"]
      Frontend["Frontend Task\n(curl http://backend.demo.local)"]
    end

    Logs["CloudWatch Logs"]
  end

  Backend -->|"create-service --service-registries"| CloudMap
  CloudMap -->|"A 记录 backend.demo.local\n(TTL=10, 健康检查)"| Frontend
  Frontend -->|"DNS 解析 + curl"| Backend
  Backend -->|"日志"| Logs
  Frontend -->|"日志"| Logs
```

**服务发现与故障恢复时序**（体现动态注册/注销，而非静态拓扑）：

```mermaid
sequenceDiagram
  participant BE as Backend Service
  participant CM as Cloud Map
  participant FE as Frontend Task

  BE->>CM: create-service 时注册 (desiredCount=2)
  CM-->>BE: 2 个实例注册成功 (IP:Port)
  FE->>CM: 解析 backend.demo.local
  CM-->>FE: 返回 2 个健康实例 IP
  FE->>BE: curl http://backend.demo.local/
  BE-->>FE: 200 backend-v1

  Note over BE,CM: 故障场景：缩容 desiredCount=0
  BE->>CM: 任务停止，自动注销实例
  CM-->>FE: 实例列表为空
  FE->>CM: 再次解析 backend.demo.local
  CM-->>FE: NXDOMAIN / 无可用实例
  FE--xBE: curl 失败 BACKEND UNREACHABLE

  Note over BE,CM: 恢复：desiredCount=2
  BE->>CM: 新任务启动，重新注册
  CM-->>FE: 实例恢复为 2 个
```

---

## Demo13 — CodeDeploy 蓝绿发布

Demo13 在独立的 `demo-ecs-bg` Service 上演示 ECS + CodeDeploy 的蓝绿部署：ALB 上并存生产 Listener（:8080）和测试 Listener（:8081），分别指向 Blue/Green 两个 Target Group。注册新 Task Definition 后提交 AppSpec，CodeDeploy 先把新任务挂到 Green 做测试流量验证，再把生产 Listener 切到 Green，原 Blue 任务延迟终止，失败时按 `auto-rollback-configuration` 自动回滚。

```mermaid
flowchart TB
  Op["实验操作者\n(AI / aws CLI)"]

  subgraph AWS["AWS us-east-1"]
    subgraph ALB_["ALB"]
      ProdListener["生产 Listener :8080"]
      TestListener["测试 Listener :8081"]
    end
    BlueTG["Target Group: Blue"]
    GreenTG["Target Group: Green"]

    subgraph Cluster["ECS Cluster: demo-ecs"]
      BGSvc["Service: demo-ecs-bg\n(deploymentController=CODE_DEPLOY)"]
      TaskV1["Task v1 (Blue)"]
      TaskV2["Task v2 (Green)"]
    end

    subgraph CD["CodeDeploy"]
      CDApp["Application: demo-ecs-app"]
      CDGroup["Deployment Group\n(BLUE_GREEN, WITH_TRAFFIC_CONTROL)"]
    end
  end

  Op -->|"1. create-target-group x2"| BlueTG
  Op -->|"1. create-target-group x2"| GreenTG
  Op -->|"2. create-listener :8081"| TestListener
  Op -->|"5. create-service"| BGSvc
  BGSvc -->|"初始挂载"| BlueTG
  BlueTG --> TaskV1
  ProdListener --> BlueTG

  Op -->|"6. create-application +\ncreate-deployment-group"| CDApp
  CDApp --> CDGroup
  CDGroup -.管控.-> BGSvc

  Op -->|"7. 注册 Task Definition v2"| TaskV2
  Op -->|"8. create-deployment (AppSpec)"| CDGroup
  CDGroup -->|"部署新任务到 Green"| GreenTG
  GreenTG --> TaskV2
  TestListener -.验证 Green.-> GreenTG
  CDGroup -->|"验证通过后切流量"| ProdListener
  ProdListener -.切换指向.-> GreenTG
  CDGroup -->|"5分钟后终止 Blue"| TaskV1
```
