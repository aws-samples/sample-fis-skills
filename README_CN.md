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
| [aws-fis-experiment-prepare](./aws-fis-experiment-prepare/) | 生成运行 AWS FIS 实验所需的所有配置文件（实验模板、IAM 策略、CFN 模板、告警、Dashboard、预期行为文档），然后通过 CloudFormation 自愈迭代部署。支持 Scenario Library 预置场景和自定义单个 FIS Action。**注意：** Scenario Library 模板（AZ Power Interruption、AZ Application Slowdown、Cross-AZ Traffic Slowdown、Cross-Region Connectivity）无法通过 API 生成 — Skill 会读取 AWS 文档提取 JSON 模板。 |
| [aws-fis-experiment-execute](./aws-fis-experiment-execute/) | 部署并运行已准备好的 AWS FIS 实验。需要一个已准备好的实验目录（来自 aws-fis-experiment-prepare），处理部署、实验启动、实时监控和清理。 |
| [eks-app-log-analysis](./eks-app-log-analysis/) | 在 FIS 故障注入实验期间或之后分析 EKS 应用日志。支持实时监控（后台日志收集 + 实时洞察）和事后分析。生成按受影响服务分组的综合报告，包含错误时间线、模式识别和恢复分析。 |

## 前置条件

本仓库中的 Skills 可能依赖以下 MCP Server 和工具：

- [**aws-knowledge-mcp-server**](https://github.com/awslabs/mcp/tree/main/src/aws-knowledge-mcp-server) — AWS 文档搜索与检索
- [**context7**](https://context7.com/) — 库和框架文档查询，提供代码示例
- **AWS CLI** — 用于可选的线上资源审计

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

