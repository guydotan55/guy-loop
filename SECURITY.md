# Security

## Reporting

Report vulnerabilities privately through GitHub's "Report a vulnerability" (Security tab of this repository). Please do not open a public issue for them.

## Threat model

guy-loop is instructions only (no scripts). Its last step hands the build to Codex through [codex-use](https://github.com/guydotan55/codex-use), which runs Codex with `--dangerously-bypass-approvals-and-sandbox`: no approval prompts and no sandbox. Read the Safety section of the [README](README.md#safety) and codex-use's own warning before use, and do not run the loop on untrusted repositories, specs or plans.
