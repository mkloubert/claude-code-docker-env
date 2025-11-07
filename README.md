# Claude Code Docker Environment Template

> A secure, containerized environment template for running [Claude Code](https://www.claude.com/product/claude-code) with network restrictions and resource controls.

## Overview

This template provides a production-ready Docker environment for safely running Claude Code in an isolated container with comprehensive security hardening, including:

- **Network-level security** with iptables firewall restricting outbound connections
- **Resource limits** to prevent system abuse
- **Dropped kernel capabilities** for minimal attack surface
- **Pre-configured development tools** for a complete coding environment

Perfect for running AI-assisted development in a controlled, auditable environment where you need to ensure Claude Code can only communicate with approved services.

---

## Features

- **Secure by Default**: Firewall-restricted network access to only essential services
- **Resource Controlled**: CPU and memory limits to prevent resource exhaustion
- **Fully Isolated**: Runs as non-root user with dropped capabilities
- **Developer Ready**: Includes git, zsh, vim, nano, fzf, and other essential tools
- **Claude Code Integrated**: Latest version of Claude Code CLI pre-installed
- **Persistent History**: Command history persists across container restarts
- **Configurable Timezone**: Set your local timezone for accurate timestamps

### Allowed Network Access

The firewall permits connections only to:
- **GitHub** (API, web, and git operations)
- **npm Registry** (package installation)
- **Anthropic API** (Claude Code authentication and API calls)
- **VSCode Marketplace** (extension downloads)
- **Statsig** (telemetry, if enabled)
- **Sentry** (error reporting)
- **Host Network** (for Docker operations)

All other outbound connections are **blocked by default**.

---

## Prerequisites

- **Docker** (version 20.10 or higher)
- **Docker Compose** (version 2.0 or higher)
- **Linux host** (for iptables support) or Docker Desktop with Linux containers

**Note**: The firewall configuration requires Linux kernel features (iptables, ipset). Windows/macOS users should use Docker Desktop with WSL2/Linux VM.

---

## Quick Start

### 1. Clone or Use This Template

```bash
# Use as GitHub template
# Click "Use this template" button on GitHub, or:

# Clone directly into your project
git clone https://github.com/mkloubert/claude-code-docker-env my-project
cd my-project
```

### 2. Copy Files to Your Project

If you have an existing project, copy these files:

```bash
cp Dockerfile.claude-code /path/to/your/project/
cp docker-compose.claude-code.yaml /path/to/your/project/
cp init-firewall.sh /path/to/your/project/
```

### 3. Start the Environment

```bash
docker compose -f docker-compose.claude-code.yaml up --build -d
docker compose -f docker-compose.claude-code.yaml exec dev zsh -c "claude"
```

This will:
- Builds and start the container in background
- Runs Claude Code in a [zsh shell](https://en.wikipedia.org/wiki/Z_shell)

### 4. Stop the container

```bash
docker compose -f docker-compose.claude-code.yaml down
```

---

## Configuration

### Customize Claude Code Version

Edit `docker-compose.claude-code.yaml` or set build arg:

```yaml
services:
  dev:
    build:
      args:
        CLAUDE_CODE_VERSION: "1.2.3"  # Specify version
```

Or use `latest` (default) for the newest version.

### Set Timezone

```yaml
services:
  dev:
    build:
      args:
        TZ: "Europe/Berlin"  # Your timezone
```

### Adjust Resource Limits

Edit resource limits in `docker-compose.claude-code.yaml`:

```yaml
deploy:
  resources:
    limits:
      cpus: '4.0'      # Maximum CPU cores
      memory: 8g       # Maximum RAM
    reservations:
      cpus: '1.0'      # Reserved CPU
      memory: 1g       # Reserved RAM
```

### Add More Allowed Domains

Edit `init-firewall.sh` and add domains to the loop at line 67:

```bash
for domain in \
    "registry.npmjs.org" \
    "api.anthropic.com" \
    "your-custom-domain.com" \
    "another-domain.com"; do
```

### User ID Mapping (Optional)

To match host user permissions, uncomment in `docker-compose.claude-code.yaml`:

```yaml
build:
  args:
    UID: ${UID-1000}
    GID: ${GID-1000}

user: "${UID-1000}:${GID-1000}"
```

---

## Usage

### Running Claude Code

Once inside the container:

```bash
# Start Claude Code
claude

# Or run specific commands
claude --help
```

### Using Development Tools

The container includes:

```bash
# Git operations
git status
gh pr list

# File editing
nano file.txt
vim file.txt

# Fuzzy finding
fzf

# Better git diffs
git diff  # Uses delta by default
```

### Persisting Command History

Your bash/zsh history is automatically persisted in `/commandhistory/.bash_history`.

### Stopping the Container

```bash
# Exit the shell
exit

# Or press Ctrl+D
```

---

## Project Structure

```
.
├── Dockerfile.claude-code            # Docker image definition
├── docker-compose.claude-code.yaml   # Docker Compose configuration
├── init-firewall.sh                  # Firewall initialization script
├── README.md                         # This file
└── .gitignore                        # Git ignore rules
```

---

## Security Features

### Kernel Capabilities

All capabilities are dropped except what's minimally required:

```yaml
cap_drop:
  - ALL
```

### Security Options

```yaml
security_opt:
  - no-new-privileges:true
```

Prevents privilege escalation within the container.

### Network Restrictions

The firewall script (`init-firewall.sh`):

1. **Flushes all existing rules** (except Docker DNS)
2. **Creates an IP allowlist** using ipset
3. **Resolves allowed domains** to IPs dynamically
4. **Aggregates IP ranges** for efficiency
5. **Blocks everything else** by default
6. **Verifies configuration** (tests access to allowed/blocked domains)

### Non-Root User

Container runs as `node` user (UID 1000) by default, not root.

---

## Troubleshooting

### Cannot Connect to Required Service

If Claude Code or another tool can't reach a necessary service:

1. Identify the blocked domain from error messages
2. Add it to `init-firewall.sh` in the domain loop
3. Rebuild the container: `docker compose -f docker-compose.claude-code.yaml build`
4. Re-run the firewall script inside the container

### Firewall Script Fails

Check that your Docker host supports:
- iptables
- ipset
- Network capabilities (CAP_NET_ADMIN)

You may need to run the container with:

```yaml
cap_add:
  - NET_ADMIN
```

Though this reduces security isolation.

### Permission Denied Errors

If you encounter permission issues with mounted files:

1. Match user IDs (see "User ID Mapping" in Configuration)
2. Or adjust file permissions on the host: `chmod -R 755 your-project/`

### Container Won't Start

```bash
# View logs
docker compose -f docker-compose.claude-code.yaml logs

# Or run with increased verbosity
docker compose -f docker-compose.claude-code.yaml up
```

---

## Advanced Configuration

### Disable Network Completely

For fully offline development (Claude Code won't work):

```yaml
network_mode: none
```

### Custom Dockerfile Modifications

You can extend the Dockerfile to add project-specific tools:

```dockerfile
# Add after USER node

# Install Python
RUN sudo apt-get update && sudo apt-get install -y python3 python3-pip

# Install project dependencies
COPY package.json package-lock.json ./
RUN npm install
```

### Environment Variables

Pass environment variables through Docker Compose:

```yaml
services:
  dev:
    environment:
      - NODE_ENV=development
      - API_KEY=${API_KEY}
```

---

## Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/amazing-feature`
3. Commit your changes: `git commit -m 'feat: add amazing feature'`
4. Push to the branch: `git push origin feature/amazing-feature`
5. Open a Pull Request

---

## Resources

- [Claude Code Documentation](https://docs.claude.com/claude-code)
- [Docker Documentation](https://docs.docker.com/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [iptables Tutorial](https://www.netfilter.org/documentation/)

---

## Support

For issues related to:
- **This template**: Open an issue in this repository
- **Claude Code**: Visit [Claude Code GitHub Issues](https://github.com/anthropics/claude-code/issues)
- **Docker**: Check [Docker Community Forums](https://forums.docker.com/)

---

### License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

### Copyright

_© 2025 Marcel Joachim Kloubert_
