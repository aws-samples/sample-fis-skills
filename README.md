[English](README.md) | [中文](README_CN.md)

# aws-samples/sample-fis-skills

A collection of agent skills for AWS FIS (Fault Injection Service) experiment preparation, execution, and application log analysis.

## Install

```bash
# Install all skills
npx skills add aws-samples/sample-fis-skills --skill '*'

# Install a specific skill
npx skills add aws-samples/sample-fis-skills --skill aws-fis-experiment-prepare

# List available skills
npx skills add aws-samples/sample-fis-skills --list
```

## Available Skills

| Skill | Description |
|-------|-------------|
| [aws-fis-experiment-prepare](./aws-fis-experiment-prepare/) | Generate all configuration files needed to run an AWS FIS experiment (experiment template, IAM policy, CFN template, alarms, dashboard, expected-behavior doc), then deploy via CloudFormation with self-healing iteration. Supports both Scenario Library pre-built scenarios and custom single FIS actions. **Note:** Scenario Library templates (AZ Power Interruption, AZ Application Slowdown, Cross-AZ Traffic Slowdown, Cross-Region Connectivity) cannot be generated via API — the skill reads AWS documentation to extract the JSON templates. |
| [aws-fis-experiment-execute](./aws-fis-experiment-execute/) | Deploy and run a prepared AWS FIS experiment. Expects a prepared experiment directory (from aws-fis-experiment-prepare) and handles deployment, experiment start, real-time monitoring, and cleanup. |
| [app-service-log-analysis](./app-service-log-analysis/) | Analyze application and managed service logs during or after FIS fault injection experiments. Supports real-time monitoring (background log collection + live insights) and post-hoc analysis. Generates comprehensive reports with error timelines, patterns, and recovery analysis grouped by affected services. |

## Prerequisites

Skills in this repo may depend on the following MCP servers and tools:

- [**aws-knowledge-mcp-server**](https://github.com/awslabs/mcp/tree/main/src/aws-knowledge-mcp-server) -- AWS documentation search and retrieval
- [**context7**](https://context7.com/) -- Library and framework documentation lookup with code examples
- **AWS CLI** -- for optional live resource auditing

<details>
<summary>OpenCode MCP configuration sample (<code>config.json</code>)</summary>

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

## Least Privilege Recommendations

Before running FIS experiments and EKS-related skills, we recommend setting up the following permissions using the principle of least privilege.

- **CloudFormation service role** — See [aws-fis-experiment-prepare Prerequisites](./aws-fis-experiment-prepare/README.md#create-a-cloudformation-service-role) for creating a dedicated CFN service role to deploy FIS experiment stacks with least privilege.

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

