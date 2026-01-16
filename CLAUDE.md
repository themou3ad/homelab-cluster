# Claude Code Instructions

## Kubernetes Commands

Always use the kubeconfig file from this project for kubectl commands:

```bash
KUBECONFIG=/Users/denden/dev/gurzil-lab/kubeconfig kubectl <command>
```

## Secrets

Secrets are encrypted with SOPS. Files must be named `*.sops.yaml` to be encrypted automatically.

To encrypt a new secret:
```bash
sops -e -i path/to/secret.sops.yaml
```

## Git Commits

- Follow Conventional Commits format: `type(scope): description`
  - Types: feat, fix, docs, refactor, chore, etc.
- Never mention Claude, AI, or any AI assistant in commit messages
- No Co-Authored-By lines referencing AI
