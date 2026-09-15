# Azure AI-200 — Study Notes & Lab Runbooks

My hands-on study journey for **Microsoft Certified: Azure AI Cloud Developer Associate (Exam AI-200, "Developing AI Cloud Solutions on Azure")**.

For each exam domain I work through the official Microsoft Learn labs, then consolidate everything I did into **one annotated runbook per domain**, command by command, so it doubles as a quick reference.

## Exam at a glance
- **AI-200** is a developer-focused exam (Python + Azure SDKs, containers, data/vector services).
- Domains and weights: Container hosting (20–25%), AI with data management services (25–30%), Connect & consume (20–25%), Secure/monitor/troubleshoot (20–25%).
- Study guide: https://aka.ms/AI200-StudyGuide

## Runbooks
| Domain | Runbook | Status |
|---|---|---|
| Container application hosting (ACR, App Service, sidecars) | [container-hosting-runbook.md](container-hosting-runbook.md) | Done |
| Deploy & manage apps on Azure Container Apps (deploy, manage, scale) | [container-apps-runbook.md](container-apps-runbook.md) | In progress |
| AI solutions with data management services | _coming_ | |
| Connect to and consume Azure services | _coming_ | |
| Secure, monitor, and troubleshoot | _coming_ | |

Each runbook has the exact `az` commands I ran (with inline comments), the gotchas I hit, and the cleanup steps.

## Notes
- The lab **starter files** (sample apps, Dockerfiles) are Microsoft's and live in the official repo: https://github.com/MicrosoftLearning/mslearn-azure-ai. This repo holds **my own** consolidated runbooks, not Microsoft's starter code.
- Resource names in the commands (registries, resource groups) are examples from my own study subscription. No secrets, keys, or connection strings are included.
