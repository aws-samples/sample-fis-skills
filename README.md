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
| [aws-fis-experiment-prepare](./aws-fis-experiment-prepare/) | Generate configuration files needed to run an AWS FIS experiment (CFN template with embedded experiment template, IAM role, dashboard), then deploy via CloudFormation with self-healing iteration. Supports both Scenario Library pre-built scenarios and custom single FIS actions. For AZ Power Interruption, supports **service-scoped sub-action pruning** — only includes sub-actions for user-specified services to prevent unintended blast radius. Default experiment duration is 10 minutes. |
| [aws-fis-experiment-execute](./aws-fis-experiment-execute/) | Run a prepared AWS FIS experiment. Extracts the template ID from the experiment directory name, queries FIS API for actions, discovers affected applications, starts the experiment with explicit user confirmation, monitors progress with real-time log insights, and generates a results report. |
| [app-service-log-analysis](./app-service-log-analysis/) | Analyze EKS application logs during or after FIS fault injection experiments. **Multi-cluster deep dependency discovery** — automatically discovers all EKS clusters in the target region, generates isolated kubeconfig files (never overwrites `~/.kube/config`), and deep-scans all accessible clusters in parallel (env vars, ConfigMaps, Secrets, ExternalName, etc.) for applications depending on fault-injected services. Supports real-time monitoring and post-hoc analysis with comprehensive reports. |

## Prerequisites

Skills in this repo may depend on the following MCP servers and tools:

- [**aws-knowledge-mcp-server**](https://github.com/awslabs/mcp/tree/main/src/aws-knowledge-mcp-server) -- AWS documentation search and retrieval
- [**context7**](https://context7.com/) -- Library and framework documentation lookup with code examples
- **AWS CLI** -- for optional live resource auditing
- **kubectl** -- configured with access to target EKS cluster(s), required by FIS experiment execution and application log analysis skills

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

## Acknowledgements

This project was inspired by and builds upon the following open-source work:

- [**aws-samples/sample-aws-resilience-skill**](https://github.com/aws-samples/sample-aws-resilience-skill) -- The resilience skill sample that provided key design patterns and architectural inspiration for this project.
- [**aws-samples/fis-template-library**](https://github.com/aws-samples/fis-template-library) -- SSM Automation documents and FIS experiment templates referenced in this project.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

