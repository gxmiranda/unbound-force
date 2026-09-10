# Quickstart: Validate the FullSend OpenCode Image

## Local Build

Build and load each supported platform image:

```bash
docker buildx build --platform linux/amd64 --load \
  --tag fullsend-opencode:local-amd64 \
  --file images/fullsend-opencode/Containerfile \
  images/fullsend-opencode

docker buildx build --platform linux/arm64 --load \
  --tag fullsend-opencode:local-arm64 \
  --file images/fullsend-opencode/Containerfile \
  images/fullsend-opencode
```

For each image, verify UID 998, exact versions, and excluded commands:

```bash
docker run --rm --platform linux/amd64 --entrypoint '' \
  fullsend-opencode:local-amd64 sh -ceu '
    test "$(id -u)" = "998"
    opencode --version
    uf --version
    ! command -v dewey
    ! command -v ollama
  '
```

Repeat with the arm64 image and platform.

## Static Validation

```bash
actionlint .github/workflows/fullsend-opencode-image.yml
npx --yes --package renovate renovate-config-validator renovate.json
git diff --check
```

## Published Digest Validation

After a non-PR publication, replace `<digest>` with the workflow output:

```bash
docker buildx imagetools inspect \
  ghcr.io/unbound-force/fullsend-opencode@sha256:<digest>

docker logout ghcr.io
docker pull --platform linux/amd64 \
  ghcr.io/unbound-force/fullsend-opencode@sha256:<digest>
docker pull --platform linux/arm64 \
  ghcr.io/unbound-force/fullsend-opencode@sha256:<digest>
```

Verify the signature and attestations with the repository's documented
certificate identity and the workflow's OIDC issuer.

## FullSend Harness Validation

Configure a minimal harness with:

```yaml
image: ghcr.io/unbound-force/fullsend-opencode@sha256:<digest>
```

Run the harness once for each supported platform and record the digest, image
platform, workflow run, and successful startup evidence.
