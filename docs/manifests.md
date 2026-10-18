# Writing manifests

All four kinds can live in one multi-document YAML file. See [`examples/task.yaml`](../examples/task.yaml) for a complete, working set.

`metadata.name` and `metadata.atespace` become Substrate resource names, so they must be lowercase RFC 1123 labels: at most 63 lowercase alphanumeric characters or `-`, starting and ending with an alphanumeric character. `ax apply` rejects anything else up front rather than letting the task fail later with `ActorCreationFailed`.

## Task

```yaml
apiVersion: ax.io/v1alpha1
kind: Task
metadata:
  name: task123
  atespace: default
spec:
  image: "ghcr.io/my-org/my-agent-image"
  command: ["python", "agent.py"]
  env:
    - name: ENVIRONMENT
      value: "production"

  resources:
    requests:
      cpu: "500m"
      memory: "1Gi"
    limits:
      cpu: "2"
      memory: "4Gi"

  workspace:
    name: default-workspace
    path: "/workspace"   # optional; this is the default
    goal: "Install dependencies and run the test suite"   # Antigravity prepares the workspace to this goal on first run

  debug: true   # serve guest services inside the sandbox so `ax ssh` works; off by default
```

### Binding a workspace

A task binds at most one `Workspace` through `spec.workspace`. The binding needs a `name`; without a `path` the workspace is mounted at `/workspace`. That path is the working directory of `spec.command`, and the task reports `WorkspaceReady` once the workspace is prepared. A task with no `workspace` gets an empty working directory at `/workspace`.

To work on content from several sources, put it all in one `Workspace`, for example by listing several repositories under `spec.git` or adding files through `spec.files`.

## Workspace

```yaml
apiVersion: ax.io/v1alpha1
kind: Workspace
metadata:
  name: default-workspace
  atespace: default
spec:
  git:
    - name: origin
      repo: "https://github.com/chalk/chalk.git"
      branch: "main"
  files:
    - path: "AGENTS.md"
      content: |
        # Project Guidelines
        - Run `go test ./...` before submitting changes.
        - Keep dependencies minimal.
  mcp:
    registries:
      - provider: google
        query: "mcp.tags:build"
    servers:
      - name: git-tools
        endpoint: "http://git-mcp.default.svc.cluster.local:8080"
  skills:
    registries:
      - provider: google
        query: "skills.tags:nodejs"
    path: "/.agents/skills"
```


## Model

For Google models, store the API key and set `provider: google`.

```bash
kubectl create secret generic gemini-api-secret --from-literal=GEMINI_API_KEY="AIzaSy..."
```

Then reference it from the `Model`:

```yaml
apiVersion: ax.io/v1alpha1
kind: Model
metadata:
  name: default-model
  atespace: default
spec:
  provider: google
  model: gemini-3.8-flash
  secretKey:
    name: gemini-api-secret
    key: GEMINI_API_KEY
```

For Anthropic models, store the key the same way and set `provider: anthropic`.

```bash
kubectl create secret generic anthropic-api-secret --from-literal=ANTHROPIC_API_KEY="sk-ant-..."
```

```yaml
apiVersion: ax.io/v1alpha1
kind: Model
metadata:
  name: claude-model
  atespace: default
spec:
  provider: anthropic
  model: claude-opus-5
  secretKey:
    name: anthropic-api-secret
    key: ANTHROPIC_API_KEY
  parameters:
    maxTokens: 16000
```
