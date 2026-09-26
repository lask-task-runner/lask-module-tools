# lask-module-tools

Ready-made execution environments for the tools a [Lask](https://github.com/lask-task-runner/lask) task runs. Each tool is a function that returns an `Environment`; its keyword parameters are the release of its image, and what the tool needs from the environment it runs in — the variables it reads, its credentials, and the host directories it has to see.

```lask
import * as tools from "tools"

command { "aws" } on tools.aws(profile = "dev", config_dir = tools.home(".aws"))

deploy(): String = $ aws s3 sync web/dist s3://my-bucket --delete
```

The tools themselves are not wrapped: `$ aws s3 sync ...` is already the clearest way to say it. Design and the tools planned next: [doc/design.md](doc/design.md).

Requires a Lask whose container options take `null` (lask#49) and accept a typed list or table (lask#50), and whose secrets may be `null` (lask#51).

Pull a tool's image once before the first command that uses it, e.g. `docker pull amazon/aws-cli:2.36.41`. A tool builds its image reference from `--tag` when it runs, so Lask treats the reference as computed at run time (lask spec 10.3): `lask env build` cannot pull or pin it, and a run never pulls. A tool built from a recipe in this module — marked *recipe* below — is the exception: it has no `--tag`, and `lask deps sync` or `lask env build` builds it.

## Install

```text
lask deps add tools --git https://github.com/lask-task-runner/lask-module-tools --rev <rev>
```

## Conventions

Every tool follows the same rules:

- **`--tag` picks the release of the tool's image**, and defaults to the one this module was written and tested against. Pull the image for the tag you run (above).
- **A parameter left `null` sets nothing.** Every parameter defaults to `null`, and a variable given `null` is left out. `""` is a value: it sets its variable to the empty string.
- **Secrets are `!!` parameters**, so they are masked in the command log, including in the environment it records — whatever the caller passed. They default to `null` like the rest, and a `null` secret registers nothing for masking.
- **A host path is mounted, never expanded.** Lask starts `docker` without a shell, so `~` would reach the daemon as a directory named `~`. Use `tools.home(".aws")`.
- **`--with: Array<ToolSetup>`** carries another tool's variables and mounts, so the Terraform AWS provider can run with the same account as the AWS CLI: `terraform(with = [tools.aws_setup(profile = "dev")])`.
- **`--extra_env: Map<String>`** sets any variable the tool has no parameter for.
- **Precedence**, later wins: the tool's defaults, `--extra_env`, each setup in `--with`, the tool's own named parameters.

## Catalog

Every function returns an `Environment`. A tool that works as it is exports its command words, which a project imports by name (`import command { "go" } from "tools"`); a tool that needs a credential, a cluster or a server exports none, and the project declares its own (`command { "kubectl" } on tools.kubectl(...)`). Every image has `/bin/sh`, which a Lask command runs through.

### Cloud CLIs

| Function | Image, default | Command words |
|---|---|---|
| `aws` | `amazon/aws-cli:2.36.41` | — |
| `gcloud` | `gcr.io/google.com/cloudsdktool/google-cloud-cli:586.0.0-slim` | — |
| `az` | `mcr.microsoft.com/azure-cli:2.90.0` | — |

### Languages

| Function | Image, default | Command words |
|---|---|---|
| `python` | `python:3.12.14-alpine3.24` | `python`, `python3`, `pip`, `pip3` |
| `node` | `node:24.21.0-alpine3.24` | `node`, `npm`, `npx`, `corepack` |
| `go` | `golang:1.27.1-trixie` | `go`, `gofmt` |
| `java` | `eclipse-temurin:25.0.4.1_1-jdk-noble` | `java`, `javac`, `jar` |
| `maven` | `maven:3.9.16-eclipse-temurin-25-noble` | `mvn` |
| `gradle` | `gradle:9.8.0-jdk25-noble` | `gradle` |
| `rust` | `rust:1.98.1-slim-trixie` | `cargo`, `rustc`, `rustup` |
| `cc` | `gcc:16.2.0` | `gcc`, `g++`, `cc`, `c++`, `make` |
| `ruby` | `ruby:4.0.7-alpine3.24` | `ruby`, `gem`, `bundle`, `rake` |
| `dotnet` | `mcr.microsoft.com/dotnet/sdk:10.0.401-alpine3.24` | `dotnet` |
| `deno` | `denoland/deno:alpine-2.9.7` | `deno` |
| `bun` | `oven/bun:1.4.2-alpine` | `bun`, `bunx` |
| `haskell` | `haskell:9.12.4-slim-bookworm` | `ghc`, `ghci`, `runghc`, `cabal`, `stack` |

### Container orchestration

| Function | Image, default | Command words |
|---|---|---|
| `kubectl` | `alpine/k8s:1.37.0` | — |
| `helm` | `alpine/k8s:1.37.0` | — |
| `kustomize` | `alpine/k8s:1.37.0` | `kustomize` |

### IaC

| Function | Image, default | Command words |
|---|---|---|
| `terraform` | `hashicorp/terraform:1.16.4` | — |
| `tofu` | `ghcr.io/opentofu/opentofu:1.12.6` | — |
| `ansible` | *recipe*: ansible-core 2.21.4 on Python 3.12 | — |
| `packer` | `hashicorp/packer:1.16.1` | — |
| `pulumi` | `pulumi/pulumi-<language>:3.265.0` | — |

### Database clients

| Function | Image, default | Command words |
|---|---|---|
| `psql` | `postgres:18.6-alpine3.24` | — |
| `mysql` | `mysql:8.4.11` | — |
| `redis_cli` | `redis:8.10.2-alpine` | — |
| `migrate` | `migrate/migrate:v4.20.1` | — |
| `flyway` | `flyway/flyway:13.8.0-alpine` | — |

### Git and code hosts

| Function | Image, default | Command words |
|---|---|---|
| `git` | *recipe*: the image of `unix` | — |
| `gh` | *recipe*: the image of `unix` | — |
| `glab` | *recipe*: the image of `unix` | — |

### Build and code generation

| Function | Image, default | Command words |
|---|---|---|
| `buf` | `bufbuild/buf:1.73.0` | `buf` |
| `protoc` | *recipe*: protobuf and the gRPC plugins on Alpine 3.24 | `protoc` |
| `bazel` | *recipe*: Bazelisk 1.29.0 on Debian | `bazel`, `bazelisk` |

### Linters and formatters

| Function | Image, default | Command words |
|---|---|---|
| `shellcheck` | `koalaman/shellcheck-alpine:v0.11.0` | `shellcheck` |
| `hadolint` | `hadolint/hadolint:v2.15.1-alpine` | `hadolint` |
| `tflint` | `ghcr.io/terraform-linters/tflint:v0.64.0` | `tflint` |
| `golangci_lint` | `golangci/golangci-lint:v2.14.0` | `golangci-lint` |
| `ruff` | `ghcr.io/astral-sh/ruff:0.16.9-alpine` | `ruff` |
| `actionlint` | `rhysd/actionlint:1.7.12` | `actionlint` |

### Security scanners

| Function | Image, default | Command words |
|---|---|---|
| `trivy` | `aquasec/trivy:0.74.0` | `trivy` |
| `gitleaks` | `zricethezav/gitleaks:v8.30.1` | `gitleaks` |
| `checkov` | `bridgecrew/checkov:3.3.19` | `checkov` |
| `grype` | *recipe*: grype v0.119.0 and syft v1.52.0 on Alpine 3.24 | `grype` |
| `syft` | *recipe*: the image of `grype` | `syft` |
| `semgrep` | `semgrep/semgrep:1.178.0` | `semgrep` |

### Secret managers

| Function | Image, default | Command words |
|---|---|---|
| `vault` | `hashicorp/vault:1.21.4` | — |
| `sops` | `ghcr.io/getsops/sops:v3.13.3-alpine` | — |
| `op` | `1password/op:2.39.0` | — |

### Everyday tools

| Function | Image, default | Command words |
|---|---|---|
| `unix` | *recipe*: Alpine 3.24 with the tools below | `curl`, `wget`, `openssl`, `jq`, `yq`, `envsubst`, `tar`, `gzip`, `xz`, `bzip2`, `zip`, `unzip`, `rsync` |

### API, HTTP, E2E and load testing

| Function | Image, default | Command words |
|---|---|---|
| `playwright` | `mcr.microsoft.com/playwright:v1.63.0-noble` | — (its tests run through node's `npx`) |
| `k6` | `grafana/k6:1.8.1` | `k6` |
| `grpcurl` | `fullstorydev/grpcurl:v1.9.3-alpine` | `grpcurl` |
| `newman` | `postman/newman:6.1.3-alpine` | `newman` |
| `robot` | *recipe*: Robot Framework 7.5 and RequestsLibrary 0.9.7 on Python 3.12 | `robot`, `rebot`, `libdoc` |
| `ab` | `httpd:2.4.68-alpine3.24` | `ab` |

## Setups

A setup is what one tool needs from an environment, as a value another tool's `--with` takes: the Terraform AWS provider needs what the AWS CLI needs. Each `<tool>_setup` takes the parameters of its tool that describe an account; the others belong to no one tool.

| Setup | Parameters | Sets |
|---|---|---|
| `aws_setup` | `profile`, `region`, `endpoint_url`, `access_key_id`, `secret_access_key!!`, `session_token!!`, `config_dir` | `AWS_*`; mounts `~/.aws` read-only |
| `gcloud_setup` | `project`, `region`, `credentials_file`, `access_token!!`, `config_dir` | `CLOUDSDK_*` for gcloud and `GOOGLE_*` for the SDKs; mounts the credentials file read-only and `~/.config/gcloud` |
| `azure_setup` | `subscription_id`, `tenant_id`, `client_id`, `client_secret!!`, `config_dir` | `ARM_*` for Terraform and `AZURE_*` for the SDKs; mounts `~/.azure` |
| `kube_setup` | `kubeconfig`, `context`, `namespace` | `KUBECONFIG`, `KUBE_CONFIG_PATH`, `HELM_KUBECONTEXT`, `KUBE_CTX`, `HELM_NAMESPACE`; mounts the kubeconfig read-only |
| `pg_setup` | `host`, `port`, `user`, `password!!`, `database`, `sslmode` | `PG*`, which libpq, psycopg and node-postgres read |
| `vault_setup` | `addr`, `token!!`, `namespace`, `skip_verify` | `VAULT_*` |
| `ssh_setup` | `dir`, `agent_socket` | mounts the key directory read-only at `/root/.ssh`, the agent socket at `SSH_AUTH_SOCK` |
| `git_setup` | `token!!`, `host`, `token_user`, `name`, `email` | trusts the mounted project; with a token, rewrites `https://<host>/` and `git@<host>:` to https with the token; the commit identity |
| `docker_setup` | `socket` | mounts the host's Docker daemon: root on the host, so ask for it only where it is needed |
| `registry_setup` | `registry`, `username`, `password!!`, `config_dir` | each scanner's registry credentials, or a Docker configuration read-only (`DOCKER_CONFIG`) |

```lask
creds = tools.aws_setup(profile = "dev", region = "ap-northeast-1", config_dir = tools.home(".aws"))

command { "terraform" } on tools.terraform(with = [creds])
command { "kubectl", "helm" } on tools.helm(kubeconfig = tools.home(".kube/config"), with = [creds])
build(): String = $[tools.go(with = [tools.git_setup(token = get_env("GITHUB_TOKEN"))])] go build ./...
```

A setup carries none of its tool's defaults. Two setups that set the same variable follow the order of `--with`; `git_setup` sets Git's configuration as a whole, so give one.

## Parameters

Every function also takes `--with` and `--extra_env`; a function on a registry image takes `--tag`. The other parameters, by genre (the sections after this one cover the tools that need more words):

- **Cloud CLIs.** `gcloud` and `az` take the parameters of their setup. `gcloud` never prompts (`CLOUDSDK_CORE_DISABLE_PROMPTS=1`); `az` sends no telemetry, and logs in from a service principal only when told to: `az login --service-principal -u "$AZURE_CLIENT_ID" -p "$AZURE_CLIENT_SECRET" --tenant "$AZURE_TENANT_ID"`.
- **Languages.** Each takes `--cache_dir`, mounted at `/cache`, where its package cache lives: `GOMODCACHE`/`GOCACHE`, `-Dmaven.repo.local`, `GRADLE_USER_HOME`, `CARGO_HOME`, `BUNDLE_PATH`, `NUGET_PACKAGES`, `DENO_DIR`, `BUN_INSTALL_CACHE_DIR`, `CABAL_DIR`/`STACK_ROOT`. Beyond it: `go` — `goproxy`, `goprivate`, `goflags`, `cgo_enabled`, `goos`, `goarch`, and `GOTOOLCHAIN=local` so Go stays at the release `--tag` names; `java`, `maven`, `gradle` — `java_tool_options`, `maven_opts`, `gradle_opts`, and Maven in batch mode; `rust` — `rustflags`; `cc` — `cc`, `cxx`, `cflags`, `cxxflags`, `ldflags`, `makeflags`; `deno` — `auth_tokens!!`; `bun` — `npm_token!!`. Stack in the `haskell` image does not use the GHC beside it unless given `--system-ghc`.
- **Container orchestration.** `kubectl` and `helm` take the parameters of `kube_setup` and `network`; `helm` also `cache_dir`. kubectl reads no variable for a context or a namespace: give `--context` and `-n` on its command line. A cluster at 127.0.0.1 (kind, minikube) takes `network = "host"`. The image carries the AWS CLI for an EKS kubeconfig, not the GKE auth plugin.
- **IaC.** `tofu` takes the parameters of `terraform`. `packer` — `log`, `variables` (as `PKR_VAR_*`), `cache_dir`. `pulumi` — `language` (`nodejs`, `python`, `go`, `dotnet`, `java`), `access_token!!`, `backend_url`, `config_passphrase!!`, `cache_dir`. `ansible` — `config`, `inventory`, `host_key_checking`, `vault_password_file`, `ssh_dir`, `ssh_agent_socket`, `cache_dir`.
- **Database clients.** Each takes `network`, to join a Compose network. `psql` takes the parameters of `pg_setup`. `mysql` — `host`, `port`, `user`, `password!!`, `database`; the client reads no user and no database from its environment, so they are set as `MYSQL_USER` and `MYSQL_DATABASE` for the command line: `mysql -u "$MYSQL_USER" "$MYSQL_DATABASE"`. `redis_cli` — `password!!`; the host goes on the command line. `migrate` — `database_url!!`, as `DATABASE_URL` for `-database "$DATABASE_URL"`. `flyway` — `url`, `user`, `password!!`, `locations`.
- **Git and code hosts.** `git` takes the parameters of `git_setup` and of `ssh_setup` (`ssh_dir`, `ssh_agent_socket`). `gh` — `token!!`, `host`, `repo`; `glab` — `token!!`, `host`. A `gh` or `glab` command that runs git over https takes `with = [tools.git_setup(token = ...)]` too.
- **Build.** `buf` — `token!!`, `cache_dir`. `bazel` — `cache_dir`, which keeps both the Bazel releases Bazelisk downloads and Bazel's output root, so a build is incremental across runs.
- **Linters.** `shellcheck` — `opts`; `tflint` — `github_token!!`, `cache_dir`; `golangci_lint`, `ruff` — `cache_dir`.
- **Security scanners.** `trivy` — `severity`, `exit_code`, `cache_dir`; `grype` — `cache_dir`; `checkov` — `api_key!!`; `semgrep` — `app_token!!`. Without a cache, trivy and grype download their database on every run. A container image is reached through `registry_setup` or `docker_setup`.
- **Secret managers.** `vault` takes the parameters of `vault_setup`. `sops` — `age_key!!`, `age_key_file`, and a cloud KMS through `--with`. `op` — `service_account_token!!`, `connect_host`, `connect_token!!`. A value read out of one of them is not masked by Lask: bind it to a `!!` parameter where it is used.
- **Testing.** Each takes `network`. `k6` — `cloud_token!!`; `robot` — `options` (`ROBOT_OPTIONS`).

Tools that run Git in the mounted project — `go`, `rust`, `haskell`, `pulumi`, `buf`, `bazel`, `golangci_lint`, `trivy`, `gitleaks`, `checkov`, `semgrep`, `git`, `gh`, `glab` — trust it (`safe.directory`): it belongs to the host's user, the container runs as root, and Git would otherwise refuse it.

## Tools

### `aws` — AWS CLI v2

Image `amazon/aws-cli:<tag>`, `2.36.41` unless `--tag` says otherwise. Declare the command word in your module:

```lask
command { "aws" } on tools.aws(profile = "dev", region = "ap-northeast-1", config_dir = tools.home(".aws"))
```

| Parameter | Sets |
|---|---|
| `--tag` | the image: `amazon/aws-cli:<tag>`; default `2.36.41` |
| `--profile` | `AWS_PROFILE` |
| `--region` | `AWS_REGION` and `AWS_DEFAULT_REGION` — the CLI and the SDKs read different ones |
| `--endpoint_url` | `AWS_ENDPOINT_URL` — LocalStack or another AWS-compatible endpoint |
| `--access_key_id` | `AWS_ACCESS_KEY_ID` |
| `--secret_access_key!!` | `AWS_SECRET_ACCESS_KEY` |
| `--session_token!!` | `AWS_SESSION_TOKEN` |
| `--config_dir` | mounts `<dir>` read-only as `/root/.aws` |
| `--with`, `--extra_env` | as above |

Default: `AWS_PAGER=""`, so output is never held behind `less`.

`aws_setup(...)` takes the same parameters except `--with` and `--extra_env`, and returns them as a `ToolSetup` for another tool.

**Locally**, log in on the host (`aws sso login`) and pass `config_dir = tools.home(".aws")`. The mount is read-only, so an expired token fails instead of being refreshed; log in again on the host. A container cannot usefully run the login itself: it cannot open the host's browser, and its token cache would be discarded with it.

**In CI**, pass the credentials. A container sees no variable of the host unless it is given one; `find_env` gives `null` for a variable that is not set:

```lask
whoami(
  --access_key_id: String | Null = find_env("AWS_ACCESS_KEY_ID"),
  --secret_access_key!!: String | Null = find_env("AWS_SECRET_ACCESS_KEY"),
  --session_token!!: String | Null = find_env("AWS_SESSION_TOKEN")
): String = do {
  box = tools.aws(access_key_id = access_key_id, secret_access_key = secret_access_key, session_token = session_token)
  $[box] aws sts get-caller-identity
}
```

The module exports no `aws` command word. An AWS CLI with no profile and no credentials would fail at the first call, so the declaration belongs in the project that knows which account to use. Declare it once and `export` it from your own module if several of your modules need it.

More in [example/main.lask](example/main.lask).

### `python` — Python, with pip

Image `python:<tag>`, `3.12.14-alpine3.24` unless `--tag` says otherwise. Python works as it is, so its command words are exported — import the ones you run:

```lask
import command { "python", "pip" } from "tools"

test(): String = $ pip install -q -r requirements.txt && python -m unittest
```

| Parameter | Sets |
|---|---|
| `--tag` | the image: `python:<tag>`; default `3.12.14-alpine3.24` |
| `--index_url!!` | `PIP_INDEX_URL` — masked, since a private index URL often carries a credential |
| `--extra_index_url!!` | `PIP_EXTRA_INDEX_URL` |
| `--cache_dir` | mounts `<dir>` at `/cache`; `PIP_CACHE_DIR=/cache/pip` |
| `--with`, `--extra_env` | as above |

Defaults: `PYTHONUNBUFFERED=1`, so output reaches the command log as it happens; `PYTHONDONTWRITEBYTECODE=1`, so no root-owned `__pycache__` is left in the project; `PIP_DISABLE_PIP_VERSION_CHECK=1`.

Exported words: `python`, `python3`, `pip`, `pip3`, on `python()` at its defaults. For another release or a cache, declare the words yourself: `command { "python", "pip" } on tools.python(tag = "3.13.5-alpine3.22")`.

### `node` — Node.js, with npm and npx

Image `node:<tag>`, `24.21.0-alpine3.24` (an LTS release) unless `--tag` says otherwise. Its command words are exported too:

```lask
import command { "node", "npm", "npx" } from "tools"

build(): String = $ cd web && npm ci && npm run build
```

| Parameter | Sets |
|---|---|
| `--tag` | the image: `node:<tag>`; default `24.21.0-alpine3.24` |
| `--node_env` | `NODE_ENV` |
| `--node_options` | `NODE_OPTIONS` |
| `--registry` | `npm_config_registry` |
| `--npm_token!!` | `NPM_TOKEN`, for an `.npmrc` that reads `${NPM_TOKEN}` |
| `--cache_dir` | mounts `<dir>` at `/cache`; `npm_config_cache=/cache/npm` |
| `--with`, `--extra_env` | as above |

Default: `npm_config_update_notifier=false`, so npm prints no update notice into the command log.

Exported words: `node`, `npm`, `npx`, `corepack`, on `node()` at its defaults. Declare them yourself for another release: `command { "node", "npm", "npx" } on tools.node(tag = "22.23.3-alpine3.24")`.

A command string runs in one environment, so `$ python … && node …` is a conflict: split it, or give the environment with `$[...]`.

### `terraform` — Terraform

Image `hashicorp/terraform:<tag>`, `1.16.4` unless `--tag` says otherwise. A provider's credentials come in through `--with`:

```lask
command { "terraform" } on tools.terraform(
  workspace = "dev",
  cache_dir = tools.home(".terraform.d/plugin-cache"),
  with = [tools.aws_setup(profile = "dev", region = "ap-northeast-1", config_dir = tools.home(".aws"))]
)

plan(): String = $ terraform -chdir=infra plan
```

| Parameter | Sets |
|---|---|
| `--tag` | the image: `hashicorp/terraform:<tag>`; default `1.16.4` |
| `--workspace` | `TF_WORKSPACE` |
| `--log` | `TF_LOG`, e.g. `DEBUG` |
| `--variables` | `TF_VAR_<name>` for each entry of the map |
| `--cache_dir` | mounts `<dir>` at `/cache`; `TF_PLUGIN_CACHE_DIR=/cache` |
| `--with`, `--extra_env` | as above |

Defaults: `TF_INPUT=0`, so nothing waits on a prompt, and `TF_IN_AUTOMATION=1`, so Terraform leaves out the hints meant for a person at a terminal.

The plugin cache is the mount point itself, not a directory below it: `terraform init` reports an error for a cache directory that does not exist yet, and the mount point always does.

Secret input variables go in `--variables`, never on the command line. Bind the value with `!!` where it comes from, and it is masked in the command log:

```lask
apply(--db_password!!: String = get_env("DB_PASSWORD")): String = do {
  tf = tools.terraform(variables = {"db_password": db_password}, with = [tools.aws_setup(profile = "dev")])
  $[tf] terraform -chdir=infra apply -auto-approve
}
```

The module exports no `terraform` command word, for the reason `aws` exports none: a provider without credentials fails at the first plan.

### `playwright` — Playwright, with its browsers

Image `mcr.microsoft.com/playwright:<tag>`, `v1.63.0-noble` unless `--tag` says otherwise. Pick the tag of the `@playwright/test` version the project installs: the image carries one release's browsers, and they are never downloaded here, so a mismatch fails at once.

```lask
e2e(--base_url: String = "http://host.docker.internal:3000"): String =
  $[tools.playwright(extra_env = {"E2E_BASE_URL": base_url})] cd e2e && npm ci && npx playwright test
```

| Parameter | Sets |
|---|---|
| `--tag` | the image: `mcr.microsoft.com/playwright:<tag>`; default `v1.63.0-noble` |
| `--shm_size` | the container's `/dev/shm`; default `"1g"`, since Chromium runs out of Docker's 64MB; `null` leaves Docker's |
| `--registry` | `npm_config_registry` |
| `--npm_token!!` | `NPM_TOKEN`, for an `.npmrc` that reads `${NPM_TOKEN}` |
| `--cache_dir` | mounts `<dir>` at `/cache`; `npm_config_cache=/cache/npm` |
| `--with`, `--extra_env` | as above |

Defaults: `PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1`, `npm_config_update_notifier=false`, and an init process that reaps the browsers' child processes.

The module exports no command word for it: the tests run through `npx`, which is `node`'s. Give the environment with `$[...]` as above.

### `unix` — curl, jq and the everyday command-line tools

One image with bash, curl, wget, openssl, jq, yq, envsubst, GNU coreutils, findutils, grep, sed, awk, diff, file, tar, gzip, xz, bzip2, zip, unzip, rsync, make and the SSH client — and git, gh and glab, which `git`, `gh` and `glab` configure. A command string runs in one environment, so a pipeline such as `curl … | jq …` needs both in one image: this is that image.

```lask
import command { "curl", "jq" } from "tools"

latest(): String = $ curl -fsSL https://api.github.com/repos/hashicorp/terraform/releases/latest | jq -r .tag_name
```

It is built from a recipe in this module, `lib/images/unix/Dockerfile` (Alpine 3.24, pinned by its digest), so it has no `--tag`, and `lask deps sync` or `lask env build` builds it once per machine. Its parameters are `--with` and `--extra_env` — `extra_env = {"HTTPS_PROXY": "..."}` behind a proxy.

Exported words: `curl`, `wget`, `openssl`, `jq`, `yq`, `envsubst`, `tar`, `gzip`, `xz`, `bzip2`, `zip`, `unzip`, `rsync`, on `unix()`. The shell's own words (`grep`, `sed`, `awk`, `sort`, …) are in the image but not exported: every image has them, and importing them would make a string such as `npm test | grep ok` conflict. They run here whenever an exported word selects this environment. Nor are `git`, `gh`, `glab` and `ssh`, which need an identity, or `make`, which is `cc`'s.

## Layout

Each tool, or each family of tools, lives in `lib/<name>.lask`, built from `lib/common.lask`, and `main.lask` re-exports its public functions. A recipe lives in `lib/images/<name>/`, since a Dockerfile must be inside the tree of the module that names it. Only what `main.lask` lists reaches a project, and the command line of this repository reaches each of them as well: `lask eval aws --profile dev`, `lask run aws --help`.

## Testing

```text
lask check                                       # the module
lask check --module example/main.lask            # a project using it
lask eval --module test/selftest.lask all        # every environment, compared exactly; no Docker
lask env build --module test/selftest.lask       # pulls and pins the images the selftest names, builds the recipes
lask eval --module test/selftest.lask smoke      # runs each tool; needs Docker
```
