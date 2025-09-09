# Build & Push to ECR (Composite Action)

Builds a Docker image (Linux or Windows) and pushes it to Amazon ECR using OIDC‑assumed AWS credentials.

## Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `dockerfile` | Yes | — | Path to the Dockerfile. |
| `ecr-repo` | Yes | — | ECR repository name (e.g. `<company>/<repo>`). Must already exist. |
| `iam-role` | Yes | — | AWS IAM Role ARN assumable via GitHub OIDC. |
| `context` | No | `.` | Build context directory passed to `docker build`. |
| `image-tag` | No | (auto short SHA) | Explicit image tag; if omitted a short commit SHA is used. |
| `aws-region` | No | `eu-west-1` | AWS region for ECR. |
| `platform` | No | `linux` | `linux` or `windows`. Runner OS must match. |
| `provenance` | No | `false` | Passed to `docker/build-push-action` (Linux only). |

## Output

| Name | Description |
|------|-------------|
| `image-uri` | Fully qualified image URI (e.g. `11111111111.dkr.ecr.eu-west-1.amazonaws.com/company/repo:abc1234`). |

## Usage (Linux – default)

```yaml
name: Build Linux Image
on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: ./.github/actions/build-and-push-ecr
        id: build
        with:
          dockerfile: <path/to/Dockerfile>
          ecr-repo: <company>/<repo>
          iam-role: <AWS_OIDC_ROLE_ARN> # Preferably as ${{ secrets.AWS_OIDC_ROLE_ARN }}
      - name: Use image URI
        run: echo "Pushed -> ${{ steps.build.outputs.image-uri }}"
```

## Usage (Windows image)

```yaml
name: Build Windows Image
on: workflow_dispatch

jobs:
  build:
    runs-on: windows-latest
    steps:
      - uses: ./.github/actions/build-and-push-ecr
        id: build
        with:
          platform: windows
          dockerfile: <path/to/Dockerfile>
          ecr-repo: <company>/<repo>
          iam-role: <AWS_OIDC_ROLE_ARN> # Preferably as ${{ secrets.AWS_OIDC_ROLE_ARN }}
      - name: Output
        shell: powershell
        run: echo "Image = ${{ steps.build.outputs.image-uri }}"
```

## Notes

- The caller workflow must choose `runs-on` matching `platform`.
- Windows builds use classic `docker build`; multi-arch (linux+windows) requires separate workflows plus a manifest merge (not included).
- Ensure the ECR repository exists (create separately via IaC or AWS CLI).
- `image-tag` override is optional; omit for automatic short SHA.
- `provenance` only applies to Linux builds (silently ignored on Windows path).

## Troubleshooting

| Issue | Cause / Fix |
|-------|-------------|
| Auth failure | Check IAM role trust policy includes GitHub OIDC provider + repo. |
| Repo not found | Create ECR repo or correct `ecr-repo` name. |
| Windows build fails with base image mismatch | Ensure Dockerfile uses a Windows base matching `windows-latest` (e.g. `ltsc2022`). |
| Tag not as expected | Provide `image-tag` input explicitly. |

## Example: Custom tag

```yaml
- uses: ./.github/actions/build-and-push-ecr
  with:
    dockerfile: <path/to/Dockerfile>
    ecr-repo: <company>/<repo>
    iam-role: <AWS_OIDC_ROLE_ARN> # Preferably as ${{ secrets.AWS_OIDC_ROLE_ARN }}
    image-tag: v1.4.0
```