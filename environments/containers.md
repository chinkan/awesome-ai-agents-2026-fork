# Container patterns for agent isolation

Containers are the most common isolation mechanism for running agent workloads in production. They're well-understood, widely supported, and fast enough for most use cases. This page covers patterns for using containers specifically in agentic workflows.

## Why containers for agents

Containers give you process-level isolation with namespace and cgroup boundaries. They're not as secure as microVMs (a container escape is possible, a VM escape is much harder), but they're:

- **Fast**: ~500ms cold start, near-zero warm start
- **Portable**: Run anywhere Docker runs
- **Well-tooled**: Extensive ecosystem for building, deploying, and monitoring
- **Familiar**: Most teams already know Docker

For agents running trusted code (your code, not agent-generated code), containers are usually sufficient. For agents running untrusted code, consider adding gVisor or running inside a microVM.

## Container tools

**[Docker (Moby)](https://github.com/moby/moby)** — Runs agent workloads in namespace-isolated containers. Cold start: ~500ms. Isolation: namespace + cgroup. SDK: All languages. Tags: `sandbox` `docker`

**[Podman](https://github.com/containers/podman)** — Runs rootless, daemonless containers for better security. Cold start: ~500ms. Isolation: rootless container. SDK: All languages. Tags: `sandbox` `docker`

**[Sysbox](https://github.com/nestybox/sysbox)** — Runs Docker-in-Docker securely, allowing agents to spin up their own containers. Cold start: ~1s. Isolation: enhanced container with syscall interception. SDK: All languages. Tags: `sandbox` `docker`

**[Testcontainers](https://github.com/testcontainers/testcontainers-python)** — Spins up disposable containers for testing agent tool interactions. Cold start: ~1-2s. Isolation: standard container. SDK: Python, Java, Go, .NET. Tags: `sandbox` `docker` `python`

## Patterns

### Pattern 1: One container per agent invocation

The simplest pattern. Each time your agent runs, spin up a fresh container, execute the workflow, and tear it down. This guarantees clean state between invocations.

```yaml
# docker-compose.yml
services:
  agent-runner:
    image: agent-sandbox:latest
    mem_limit: 512m
    cpus: 1.0
    network_mode: none        # No network access by default
    read_only: true           # Read-only filesystem
    security_opt:
      - no-new-privileges     # Prevent privilege escalation
    tmpfs:
      - /tmp:size=100m        # Writable temp directory with size limit
```

### Pattern 2: Sidecar sandbox

The agent orchestrator runs in one container, and agent-generated code runs in a separate "sidecar" container. Communication happens over a local socket or HTTP. This isolates untrusted code without the overhead of a remote sandbox API.

```
┌─────────────────────────────┐
│  Pod / compose service      │
│                             │
│  ┌─────────┐  ┌──────────┐ │
│  │ Agent   │  │ Sandbox  │ │
│  │ logic   │──│ (no net) │ │
│  │ (Flask) │  │ (exec)   │ │
│  └─────────┘  └──────────┘ │
└─────────────────────────────┘
```

### Pattern 3: Pre-warmed container pool

Cold starts add latency. Keep a pool of idle containers ready to accept agent tasks. When a task arrives, assign it to a warm container; when it finishes, reset the container and return it to the pool.

```python
class ContainerPool:
    def __init__(self, image: str, pool_size: int = 5):
        self.available = []
        for _ in range(pool_size):
            container = docker.run(image, detach=True)
            self.available.append(container)

    def execute(self, code: str) -> str:
        container = self.available.pop()
        result = container.exec_run(f"python -c '{code}'")
        container.exec_run("rm -rf /tmp/*")  # Reset state
        self.available.append(container)
        return result.output
```

### Pattern 4: Docker-in-Docker for agent self-management

Some agents need to create their own containers — e.g., a DevOps agent that builds and tests Dockerfiles. Use Sysbox to allow secure Docker-in-Docker without privileged mode.

```bash
# Run agent container with Sysbox runtime (no --privileged needed)
docker run --runtime=sysbox-runc -it agent-devops:latest
# Inside the container, the agent can run:
# docker build -t test .
# docker run test pytest
```

## Security hardening checklist

- [ ] Run as non-root user (`USER 1000` in Dockerfile)
- [ ] Use `--read-only` filesystem where possible
- [ ] Set memory and CPU limits (`--memory=512m --cpus=1`)
- [ ] Disable network access for code execution (`--network=none`)
- [ ] Drop all capabilities (`--cap-drop=ALL`)
- [ ] Prevent privilege escalation (`--security-opt=no-new-privileges`)
- [ ] Use seccomp profiles to restrict syscalls
- [ ] Set execution timeout (kill container after N seconds)
- [ ] Use rootless mode (Podman) for defense-in-depth
- [ ] Consider gVisor runtime for additional syscall filtering
```
