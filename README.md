# Docker Builder & Pusher Action

A reusable GitHub Action for building and pushing Docker images to GitHub Container Registry with multi-platform support.

## Features

- 🐳 Builds multi-platform Docker images (amd64/arm64)
- 📦 Pushes to GitHub Container Registry
- 🔍 Validates builds on non-main branches (build without push)
- 🏷️ Automatic tagging with commit SHA
- 📌 Optional :latest tag on main branch
- ✨ Support for additional custom tags
- 🔄 Weekly scheduled builds support
- ☸️ Optional Kubernetes deployment integration
- 🔐 Vault integration for secure credential management

## Usage

### Basic Example

```yaml
name: Build & Push Docker Image

on:
  push:
    branches: [ "*" ]
  pull_request:
    branches: [ "*" ]
  schedule:
    - cron: '0 12 * * 0'  # Weekly build every Sunday at 12:00 UTC

jobs:
  docker-build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build and Push Docker Image
        uses: cfindlayisme/docker-builder@v1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          repository: ${{ github.repository }}
          ref: ${{ github.ref }}
```

### Advanced Example with Custom Tags

```yaml
name: Build & Push Docker Image

on:
  push:
    branches: [ "*" ]
    tags: [ "v*.*.*" ]

jobs:
  docker-build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Prepare version tag
        id: version
        run: |
          if [[ "${{ github.ref }}" == refs/tags/v* ]]; then
            VERSION=${GITHUB_REF#refs/tags/v}
            echo "tags=v$VERSION,stable" >> $GITHUB_OUTPUT
          else
            echo "tags=" >> $GITHUB_OUTPUT
          fi
      
      - name: Build and Push Docker Image
        uses: cfindlayisme/docker-builder@v1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          repository: ${{ github.repository }}
          ref: ${{ github.ref }}
          docker-platforms: 'linux/amd64,linux/arm64,linux/arm/v7'
          push-latest: 'true'
          additional-tags: ${{ steps.version.outputs.tags }}
```

### Custom Dockerfile Location

```yaml
- name: Build and Push Docker Image
  uses: cfindlayisme/docker-builder@v1
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    repository: ${{ github.repository }}
    ref: ${{ github.ref }}
    dockerfile-path: './docker/Dockerfile'
    docker-context: '.'
```

### With Kubernetes Deployment

```yaml
name: Build, Push & Deploy

on:
  push:
    branches: [ "main" ]

jobs:
  docker-build-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build, Push and Deploy
        uses: cfindlayisme/docker-builder@v1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          repository: ${{ github.repository }}
          ref: ${{ github.ref }}
          # Enable Kubernetes deployment
          deploy-enabled: 'true'
          vault-addr: ${{ secrets.VAULT_ADDR }}
          vault-role-id: ${{ secrets.VAULT_ROLE_ID }}
          vault-secret-id: ${{ secrets.VAULT_SECRET_ID }}
          deployment-name: 'my-app'
          staging-enabled: 'true'
          production-enabled: 'true'
          staging-namespace: 'staging'
          production-namespace: 'production'
```

## Inputs

### Basic Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `github-token` | GitHub token for container registry | Yes | - |
| `repository` | GitHub repository (owner/name) | Yes | - |
| `ref` | Git reference being built | Yes | - |
| `docker-platforms` | Docker platforms to build for | No | `linux/amd64,linux/arm64` |
| `dockerfile-path` | Path to Dockerfile | No | `./Dockerfile` |
| `docker-context` | Docker build context | No | `.` |
| `push-latest` | Push :latest tag on main branch | No | `true` |
| `additional-tags` | Additional tags (comma-separated) | No | `''` |

### Kubernetes Deployment Inputs (Optional)

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `deploy-enabled` | Enable Kubernetes deployment | No | `false` |
| `vault-addr` | Vault server address | No* | `''` |
| `vault-role-id` | Vault AppRole role ID | No* | `''` |
| `vault-secret-id` | Vault AppRole secret ID | No* | `''` |
| `vault-kubeconfig-path` | Vault path to kubeconfig secret | No | `kv/data/pipeline/k3s` |
| `staging-enabled` | Enable staging deployment | No | `true` |
| `production-enabled` | Enable production deployment | No | `true` |
| `staging-namespace` | Kubernetes namespace for staging | No | `staging` |
| `production-namespace` | Kubernetes namespace for production | No | `production` |
| `deployment-name` | Kubernetes deployment name | No* | `''` |

\* Required when `deploy-enabled` is `true`

## Outputs

| Output | Description |
|--------|-------------|
| `image-tags` | Docker image tags that were built/pushed |
| `deploy-result` | Deployment result (only when deploy-enabled is true) |

## Behavior

### Non-Main Branches
- Builds Docker image for validation
- Does NOT push to registry
- Tests that the image builds successfully

### Main Branch
- Builds Docker image
- Pushes to GitHub Container Registry
- Tags with commit SHA (always)
- Tags with `:latest` (if `push-latest: 'true'`)
- Tags with additional custom tags (if provided)

### Tagging Strategy

Every build is tagged with the short commit SHA:
- `ghcr.io/owner/repo:abc1234`

On main branch with `push-latest: 'true'`:
- `ghcr.io/owner/repo:latest`

With additional tags (e.g., `additional-tags: 'v1.0.0,stable'`):
- `ghcr.io/owner/repo:v1.0.0`
- `ghcr.io/owner/repo:stable`

## Scheduled Builds

The action works great with scheduled builds for regular security updates:

```yaml
on:
  push:
    branches: [ "main" ]
  schedule:
    - cron: '0 12 * * 0'  # Every Sunday at 12:00 UTC
```

This ensures your Docker images are rebuilt regularly with the latest base image updates.

## Kubernetes Deployment

When `deploy-enabled: 'true'`, the action will:

1. **Retrieve kubeconfig from Vault**: Uses HashiCorp Vault with AppRole authentication to securely fetch Kubernetes credentials
2. **Deploy to Staging**: Restarts the deployment in the staging namespace (configurable)
3. **Deploy to Production**: On main branch only, restarts the deployment in production namespace (configurable)

### Deployment Behavior

- **Staging**: Deploys on all builds when `staging-enabled: 'true'`
- **Production**: Only deploys on main branch when `production-enabled: 'true'` and `ref == 'refs/heads/main'`
- Both use `kubectl rollout restart` to trigger a deployment update with the new image

### Vault Setup Required

To use Kubernetes deployment, you need:

1. HashiCorp Vault server with AppRole authentication enabled
2. Kubeconfig stored in Vault at the path specified by `vault-kubeconfig-path`
3. GitHub secrets configured:
   - `VAULT_ADDR`: Your Vault server URL
   - `VAULT_ROLE_ID`: AppRole role ID
   - `VAULT_SECRET_ID`: AppRole secret ID

## Prerequisites

### GitHub Container Registry
- Ensure your repository has access to GitHub Container Registry
- The `GITHUB_TOKEN` is automatically provided by GitHub Actions

### Multi-Platform Builds
- Uses Docker Buildx for multi-platform builds
- Default platforms: `linux/amd64,linux/arm64`
- Can be customized with `docker-platforms` input

## Example Project Structure

```
my-docker-project/
├── .github/
│   └── workflows/
│       └── docker.yml
├── Dockerfile
└── src/
    └── ...
```

## License

LGPL v3

## Author

cfindlayisme
