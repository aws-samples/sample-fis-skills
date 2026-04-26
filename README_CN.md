[English](README.md) | [中文](README_CN.md)

# aws-samples/sample-fis-skills

一组用于 AWS FIS（故障注入服务）实验准备、执行和应用日志分析的 Agent Skills。

## 安装

```bash
# 安装所有 Skills
npx skills add aws-samples/sample-fis-skills --skill '*'

# 安装单个 Skill
npx skills add aws-samples/sample-fis-skills --skill aws-fis-experiment-prepare

# 列出可用 Skills
npx skills add aws-samples/sample-fis-skills --list
```

## 可用 Skills

| Skill | 说明 |
|-------|------|
| [aws-fis-experiment-prepare](./aws-fis-experiment-prepare/) | 生成运行 AWS FIS 实验所需的配置文件（包含实验模板、IAM 角色、Dashboard 的 CFN 模板），然后通过 CloudFormation 自愈迭代部署。支持 Scenario Library 预置场景和自定义单个 FIS Action。AZ 电力中断场景支持**按服务范围裁剪子动作** — 仅包含用户指定服务的子动作，避免影响范围过大。默认实验持续时间 10 分钟。 |
| [aws-fis-experiment-execute](./aws-fis-experiment-execute/) | 运行已准备好的 AWS FIS 实验。从实验目录名提取模板 ID，通过 FIS API 查询 Actions，发现受影响的应用，经用户明确确认后启动实验，实时监控进度并展示日志洞察，生成结果报告。 |
| [app-service-log-analysis](./app-service-log-analysis/) | 在 FIS 故障注入实验期间或之后分析 EKS 应用日志。**多集群深度依赖发现** — 自动发现目标 Region 中所有 EKS 集群，为每个集群生成独立 kubeconfig 文件（绝不覆盖 `~/.kube/config`），并行深度扫描所有可访问集群（环境变量、ConfigMap、Secret、ExternalName 等）查找依赖故障注入目标服务的应用。支持实时监控和事后分析，生成综合报告。 |

## 前置条件

本仓库中的 Skills 可能依赖以下 MCP Server 和工具：

- [**aws-knowledge-mcp-server**](https://github.com/awslabs/mcp/tree/main/src/aws-knowledge-mcp-server) — AWS 文档搜索与检索
- [**context7**](https://context7.com/) — 库和框架文档查询，提供代码示例
- **AWS CLI** — 用于可选的线上资源审计
- **kubectl** — 需配置好对目标 EKS 集群的访问权限，FIS 实验执行和应用日志分析 Skills 需要

<details>
<summary>OpenCode MCP 配置示例（<code>config.json</code>）</summary>

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "aws-knowledge-mcp-server": {
      "type": "local",
      "command": ["uvx", "fastmcp", "run", "https://knowledge-mcp.global.api.aws"],
      "enabled": true
    },
    "context7": {
      "type": "remote",
      "url": "https://mcp.context7.com/mcp",
      "enabled": true,
      "headers": {
        "CONTEXT7_API_KEY": "<your-api-key>"
      }
    }
  }
}
```

</details>

## 最小权限建议

在运行 FIS 实验和 EKS 相关 Skills 之前，建议按最小权限原则配置以下权限。

- **CloudFormation 服务角色** — 参见 [aws-fis-experiment-prepare 前置条件](./aws-fis-experiment-prepare/README.md#create-a-cloudformation-service-role) 中关于创建专用 CFN 服务角色以最小权限部署 FIS 实验 Stack 的说明。

## 安全

更多信息请参见 [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications)。

## 致谢

本项目受到以下开源项目的启发，并在其基础上构建：

- [**aws-samples/sample-aws-resilience-skill**](https://github.com/aws-samples/sample-aws-resilience-skill) -- 韧性 Skill 示例项目，为本项目提供了关键的设计模式和架构灵感。
- [**aws-samples/fis-template-library**](https://github.com/aws-samples/fis-template-library) -- 本项目中引用的 SSM Automation 文档和 FIS 实验模板。

## 许可证

本项目基于 MIT-0 许可证授权。详见 LICENSE 文件。

