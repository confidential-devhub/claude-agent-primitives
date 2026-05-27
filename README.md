# Claude Code Skills & Commands

A collection of [Claude Code](https://claude.ai/code) slash commands and skills for confidential containers development workflows.

## Available Commands

Commands are custom slash commands installed in `~/.claude/commands/` and invoked with `/command-name`.

| Command | Invocation | Description |
|---|---|---|
| [cloud-api-adaptor](./cloud-api-adaptor/) | `/cloud-api-adaptor` | Set up, build, and test cloud-api-adaptor with libvirt locally (amd64, Ubuntu 24.04 podvm via mkosi) |
| [aws-podvm-ami](./aws-podvm-ami/) | `/aws-podvm-ami` | Build a debug Ubuntu podvm mkosi image (TEE_PLATFORM=amd), upload to S3, and register as an AWS AMI |
| [aws-eks-caa](./aws-eks-caa/) | `/aws-eks-caa` | Set up CAA on AWS EKS with a custom podvm AMI, run a peer-pod test workload, and tear everything down |
| [aws-test-pr](./aws-test-pr/) | `/aws-test-pr` | Test a cloud-api-adaptor GitHub PR against an existing EKS peer-pods cluster — detects what to rebuild, builds and pushes artifacts, deploys to the cluster, and runs a smoke test |

## Available Skills

Skills extend Claude Code via the Skill tool and are installed in `~/.claude/skills/`.

| Skill | Description |
|---|---|
| *(more coming soon)* | |

## Installation

### Prerequisites

- [Claude Code](https://claude.ai/code) CLI installed
- `~/.claude/commands/` directory (created automatically by Claude Code)

### Install a single command

```bash
git clone https://github.com/confidential-devhub/claude-agent-primitives
cp claude-agent-primitives/<command-name>/<command-name>.md ~/.claude/commands/
```

### Install all commands

```bash
git clone https://github.com/confidential-devhub/claude-agent-primitives
for cmd_dir in claude-agent-primitives/*/; do
    cmd_name=$(basename "$cmd_dir")
    cp "$cmd_dir/$cmd_name.md" ~/.claude/commands/
done
```

Claude Code picks up files in `~/.claude/commands/` automatically — no restart required.

### Keep commands up to date

```bash
cd claude-agent-primitives
git pull
# Re-run the copy commands above
```

## Usage

Once installed, invoke a command with its slash command in any Claude Code session:

```
/cloud-api-adaptor setup
/cloud-api-adaptor build --debug
/cloud-api-adaptor test --filter TestLibvirtCreatePeerPod

/aws-podvm-ami build
/aws-podvm-ami upload --bucket my-bucket --region us-east-2
/aws-podvm-ami all --bucket my-bucket --region us-east-2 --cleanup

/aws-eks-caa setup --ami ami-0123456789abcdef0 --region us-east-2
/aws-eks-caa test
/aws-eks-caa teardown

/aws-test-pr 3037
/aws-test-pr 3037 --caa
/aws-test-pr 3037 --rebuild-podvm
/aws-test-pr 3037 --cleanup
```

Run any command with no arguments for an interactive prompt.

## Contributing

Each command lives in its own directory:

```
<command-name>/
├── <command-name>.md   # The command file (YAML frontmatter + Markdown body)
└── README.md           # Human-readable docs for the command
```

The command file frontmatter must include `name`, `description`, `argument-hint`, and `allowed-tools`. Use an existing command as a template, then open a pull request.
