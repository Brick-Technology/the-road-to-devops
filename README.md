# 开发运营之路（The Road To DevOps）

## 背景

随着微服务的热度不断上升，开发团队都希望有一个**高性价比**和**更现代**的自动化编译和部署的工具或系统来协助众多微服务模块的开发。

## 目标

基于私有搭建Gitlab，完成DevOps全流程。

## 名词解析

1. springblade ：一个微服务应用项目

## 技术栈

<table>
  <tr>
    <td>
      <img src=".\icon\tech-stack\gitlab.svg" alt="img" style="zoom:50%;" />
    </td>
  </tr>
  <tr>
    <td align=center>
      GitLab
    </td>
  </tr>
</table>

## DevOps流程

需求分析 -> 新建分支 -> 代码编写 -> 代码请求合并 -> 代码审查 -> 代码合并 -> 代码编译 -> 代码检查 -> 单元测试 -> 检查结果上传 -> 制品生成 -> 制品上传 -> 制品部署脚本更新 ->制品部署 -> 自动化测试

### 流水线

```mermaid
flowchart 

subgraph GitLab

  subgraph springblade
    Build --> Scan
    Scan --> Package
    Package --> Upload
    Upload --> Predeploy
    Predeploy --> Deploy
    Deploy --> Downstream


  end

  subgraph K8s Agents Springblade

    Test

  end

end

Downstream --> Test

```

#### 流水线详情


## 基于私有搭建Gitlab的DevOps实践

### 架构图

```mermaid
flowchart

subgraph GitLab

    cluster-management[[cluster-management]]
    springblade[[springblade]]
    K8s-Agents-Springblade[[K8s Agents Springblade]]

end

subgraph autok3s

  subgraph K3s

      subgraph GitLab-Agent

        GitLab-Agent-Runner[(GitLab-Agent-Runner)]
        GitLab-Agent-APP[(GitLab-Agent-APP)]
      end
      subgraph GitLabRunnerGroup
          GitLabRunner-01[(GitLabRunner-01)]
          GitLabRunner-02[(GitLabRunner-02)]
          ...
        end
      subgraph springblade-app

      end

  end

end

GitLabRunner[(GitLabRunner)]
cluster-management -- 关联 --o GitLab-Agent-Runner-- 受cluster-management委托创建 --->GitLabRunnerGroup
springblade -- 借助GitLabRunner执行流水线 --- GitLabRunnerGroup
springblade -- 委托GitlabRunner触发制品配置更新 -->K8s-Agents-Springblade -- 关联 ---o GitLab-Agent-APP --受K8s Agents Springblade委托创建app环境--> springblade-app
GitLabRunner -- 发出创建GitLabRunnerGroup的指令 -->GitLab-Agent-Runner


subgraph Nexus
  Nexus-Maven[(Maven)]
  Nexus-Npm[(Npm)]
  Nexus-Raw[(Raw)]
end
subgraph Harbor
  Harbor-Library[(library)]
  Harbor-Proxy-Cache[(proxy_cache)]
  Harbor-springblade[(springblade)]
end
subgraph Minio
  Minio-RunnerCache[(buckets.gitlab-runner)]
end
SonarQube[(SonarQube)]


GitLab --用于搭建K3S上的GitLab-Agent-Runner --- GitLabRunner



GitLabRunnerGroup --用于流水线执行代码检查--- SonarQube
GitLabRunnerGroup --用于流水线中使用缓存--- Minio
GitLabRunnerGroup --用于加速流水线应用构建--- Nexus
GitLabRunnerGroup --用于加速GitLabRunner的构建--- Harbor
GitLabRunnerGroup --用于上传镜像--- Harbor

autok3s --- 用于加速服务的搭建 --o Harbor
```

### 架构搭建

#### 搭建步骤

1. 创建基础服务
    1.1. 创建Harbor
    1.2. 创建Minio
    1.3. 创建Nexus
    1.4. 创建SonarQube
2. 搭建GitLab
3. 搭建GitLab Runner并关联到GitLab
4. 搭建autoK3s，创建K3s实例
5. 在GitLab上创建项目cluster-management,并在K3s实例上创建GitLab-Agent-Runner
6. 执行cluster-management流水线，创建GitLabRunnerGroup
7. GitLab上创建项目微服务应用springblade
8. GitLab上创建项目微服务环境代理项目K8s-Agents-Springblade，并在K3s实例上创建GitLab-Agent-APP
9. 执行springblade流水线（生成新的微服务镜像并且更新新的环境配置到K8s-Agents-Springblade）

#### 架构搭建详情

1. 创建Harbor
