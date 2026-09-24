# GitHub Actions for CI/CD, not Azure DevOps Pipelines

Code and issues already live on GitHub (see `docs/agents/issue-tracker.md`). Azure DevOps Pipelines was the natural alternative given the author's Azure background, but it would mean a second platform to maintain — a separate DevOps org, PATs bridging the two systems — for no functional gain at this scale. GitHub Actions has first-party Azure deploy actions and keeps CI checks, PR review, and pipeline runs on the same platform as the code and tickets.
