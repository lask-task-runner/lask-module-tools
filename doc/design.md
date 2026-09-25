# lask-module-tools Design Document

`lask-module-tools` is a catalog of execution environments for the tools a Lask task most often runs: cloud CLIs, infrastructure tools and language toolchains. Each entry chooses an image, whose release a caller can change, and exposes, as keyword arguments, what a caller has to supply for that tool to work — its environment variables, its credentials, and the host directories it needs to see.

It replaces `lask-aws` and `lask-terraform`, which are to be retired. Their typed functions are not carried over.

Section numbers refer to the Lask specification (`lask/doc/spec.md`).

## 1. Goals

- One place where the image for each tool is chosen, so a project never has to find a working image or tag on its own — and can still run another release of it with `--tag`.
- Every environment is configured by named arguments, not by knowing which variables a tool reads: `aws(profile = "dev")`, not `env = {"AWS_PROFILE": "dev"}`.
- Environments compose. A tool that needs another tool's credentials — the Terraform AWS provider, an Ansible dynamic inventory — takes them as a value, without the caller restating them.

## 2. Non-Goals

- No wrapper functions around the tools themselves. `$ aws s3 sync ...` and `$ go test ./...` are already the clearest way to say it, and a wrapper per subcommand is an API to keep in step with every release of the tool.
- No credential provisioning. The module forwards what it is given and never runs a login flow. For SSO, log in on the host and mount the resulting directory (§5.1): a container cannot do it usefully, because it cannot open the host's browser and its token cache is discarded with it.
- No container options beyond variables and mounts in v1 (§4.5).

## 3. One Function per Tool

Every tool is a function returning an `Environment`:

```lask
aws(--tag: String = "2.36.41", --profile: String | Null = null, ...): Environment =
  #docker("amazon/aws-cli:#{tag}", ...)
```

It serves both ways of running a command, because the environment of a command declaration is an ordinary expression (ch. 5, since lask#44):

```lask
command { "aws" } on tools.aws(profile = "dev")        // then `$ aws s3 ls`
check(): String = $[tools.aws(profile = "ci")] aws sts get-caller-identity
```

The declaration's environment is evaluated when a command runs, and may not have an effect. Every function here satisfies that: it builds a value from its arguments and from `get_env`, and runs nothing.

### Command words

A module can export command words, and a project imports them by name:

```lask
import command { "go", "gofmt" } from "tools"
```

A tool **exports its command words when its default environment is useful as it is**: a language toolchain usually needs nothing, so `go`, `node` and the rest export theirs, declared on the function called with no arguments.

A tool **does not export them when it needs configuring first**. `aws` with no profile and no credentials fails at the first call, so the declaration belongs to the project that knows which account to use. A project that needs it in several modules declares it once and exports it from its own module.

## 4. Function Conventions

### 4.1 Parameters

Every parameter is a keyword parameter with a default, so every function can be called with no arguments and every argument can be given from the CLI (11.2).

| Kind | Type | Default | Meaning of the default |
| --- | --- | --- | --- |
| Image release | `--tag: String` | the release this module was tested against | The tool's image at that tag (§4.4). |
| Variable the tool reads | `String \| Null` | `null` | Not set. The variable is left out; `""` sets it to the empty string. |
| Secret | `String \| Null`, marked `!!` | `null` | Not set, and nothing registered for masking (6.10). A value is masked in the command log, including in the environment metadata, whatever the caller passed (12.3). |
| Host directory to read | `String \| Null` (a host path) | `null` | Not mounted. Mounted read-only when given. |
| Cache directory | `--cache_dir: String` | `""` | No cache. When given, mounted read-write at `/cache`, and the tool's cache variables point below it. |
| Tool-specific table | `Map<String>` | `{}` | Nothing. |
| Setups from other tools | `--with: Array<ToolSetup>` | `[]` | Nothing (§4.3). |
| Anything else | `--extra_env: Map<String>` | `{}` | Nothing. The escape hatch for a variable this module has no parameter for. |

`null` meaning "not given" is the convention of the whole module, secrets included: a parameter that was not given is `null`, and the internal `vars` helper drops the `null` entries of a variable map. `""` is a value a caller may mean — `AWS_PAGER=""` is how the pager is turned off — so it is passed as given.

A path is always a host path, and is never expanded: Lask starts `docker` without a shell, so `~` reaches the daemon as a directory named `~`. The exported `home(path)` builds one under the invoking user's home directory.

### 4.2 Precedence

Variables are layered, the later layer winning on a name present in both:

1. The function's own defaults (e.g. `AWS_PAGER=""`)
2. `--extra_env`
3. Each `ToolSetup` in `--with`, left to right
4. The function's own named parameters

Named parameters win because they are the most specific thing the caller said. `--extra_env` sits below `--with` so that it can override a default but cannot silently undo a credential a setup supplies. Mounts are concatenated in the same order.

### 4.3 Composition: `ToolSetup`

```lask
type ToolSetup = Record<env: Map<String>, volumes: Array<String>>
```

A tool whose credentials other tools need publishes a second form, `<tool>_setup(...)`, taking the same parameters as its function and returning the variables and mounts instead of an environment. The function is defined over its own setup, so the two cannot drift. A setup carries none of its tool's defaults: `AWS_PAGER` concerns the AWS CLI, not the Terraform provider.

```lask
creds = tools.aws_setup(profile = "dev", config_dir = tools.home(".aws"))

// The Terraform image, with the AWS provider's credentials.
tf = tools.terraform(variables = {"app_version": "1.2.3"}, with = [creds])
```

The same `creds` value can be handed to `ansible(with = [...])` for an AWS dynamic inventory, and a future `gcloud_setup` slots into the same list.

### 4.4 The Image Release Is `--tag`

Each tool fixes the repository of its image and takes the release as `--tag`, defaulting to the release this module was written and tested against:

```lask
aws(--tag: String = "2.36.41", ...): Environment = #docker("amazon/aws-cli:#{tag}", ...)
```

A project that needs another release says so at the call, and nothing else changes: `tools.aws(tag = "2.37.0", profile = "dev")`.

The image reference is built from the argument when the function runs, so Lask treats it as a reference computed at run time (10.3, 11.4):

- `lask envs` reports it as `<dynamic>`, and `lask env build` / `lask deps sync` neither pull nor pin it.
- A run uses the image as it is on the daemon, and never pulls it: an image that is not there is `E-IO-IMAGE-MISSING`, whose diagnostic says to pull it. A project pulls each tool's image once, `docker pull amazon/aws-cli:2.36.41`, and again when it changes `--tag`.
- The lock does not record the image's digest. The tag — an exact one, §5 — is what fixes the image, as far as the registry keeps it where it is.

A tool built from a recipe (§5.3) has no `--tag`. Its version is a build argument, and build arguments are literals (10.2), since they decide which image is built before anything runs. Its recipe is pinned the way every recipe is.

### 4.5 Container Options

A container option given `null` is left out (10.2), so a tool can take `--user: String | Null = null` and pass it straight to `#docker(...)`. v1's tools expose variables and mounts only, since none of them needs more; a tool that does — a `platform` for an image published for one architecture, a `user` for §11's root-owned files — adds the parameter and passes it through.

## 5. Catalog

| Function | Image | Exported command words | Status |
| --- | --- | --- | --- |
| `aws` | `amazon/aws-cli:2.36.41` | none (§3) | **implemented** |
| `terraform` | `hashicorp/terraform:1.16.4` | none — it needs provider credentials | **implemented** |
| `ansible` | recipe `images/ansible/Dockerfile` | none — it needs an inventory | planned |
| `go` | `golang:<tag>` | `go`, `gofmt` | planned |
| `node` | `node:24.21.0-alpine3.24` | `node`, `npm`, `npx`, `corepack` | **implemented** |
| `python` | `python:3.12.14-alpine3.24` | `python`, `python3`, `pip`, `pip3` | **implemented** |
| `java` | `eclipse-temurin:<tag>` (21, JDK) | `java`, `javac`, `jar` | planned |
| `maven` | `maven:<tag>` (3.9, Temurin 21) | `mvn` | planned |
| `gradle` | `gradle:<tag>` (8, JDK 21) | `gradle` | planned |
| `cc` | `gcc:<tag>` | `gcc`, `g++`, `cc`, `c++`, `make` | planned |

The default of each tool's `--tag` is chosen when the entry is implemented, with these rules:

- An exact tag, down to the patch release and the base-image suffix, as the default of `--tag`. The reference is computed at run time, so the lock does not pin its digest (§4.4): the exact tag is what fixes the image.
- A release still in support. In particular, not `node:20.20.2-alpine3.23`, which the `lask` examples still use: Node 20 reached end of life on 2026-04-30.
- The Ansible recipe builds from the same Python image as `python`.

### 5.1 `aws` / `aws_setup` (implemented)

| Parameter | Sets |
| --- | --- |
| `--tag` | the image, `amazon/aws-cli:<tag>`; default `2.36.41` |
| `--profile` | `AWS_PROFILE` |
| `--region` | `AWS_REGION` and `AWS_DEFAULT_REGION` — the CLI and the SDKs read different ones |
| `--endpoint_url` | `AWS_ENDPOINT_URL` — LocalStack or another AWS-compatible endpoint |
| `--access_key_id` | `AWS_ACCESS_KEY_ID` |
| `--secret_access_key!!` | `AWS_SECRET_ACCESS_KEY` |
| `--session_token!!` | `AWS_SESSION_TOKEN` |
| `--config_dir` | mounts `<dir>:/root/.aws:ro` |

Default: `AWS_PAGER=""`. The CLI would otherwise hold output behind `less`, which blocks under `lask cmd` on a terminal.

Local use is `config_dir = tools.home(".aws")` after `aws sso login` on the host. The mount is read-only, so an expired token fails rather than being refreshed: the fix is to log in again on the host. CI passes the three credentials.

The access key id is not a `!!` parameter: it identifies a key and is not a secret, and masking it would hide from the log which key a run used.

### 5.2 `terraform` (implemented)

| Parameter | Sets |
| --- | --- |
| `--tag` | the image, `hashicorp/terraform:<tag>`; default `1.16.4` |
| `--workspace` | `TF_WORKSPACE` |
| `--log` | `TF_LOG` |
| `--variables: Map<String>` | `TF_VAR_<name>` for each entry |
| `--cache_dir` | mounts at `/cache`; `TF_PLUGIN_CACHE_DIR=/cache` |

Defaults: `TF_IN_AUTOMATION=1`, `TF_INPUT=0`, so nothing ever waits on a prompt. Provider credentials come in through `--with`.

The plugin cache is the mount point itself, not a directory below it as for pip and npm. Pointed at `/cache/plugins` on an empty host directory, `terraform init` reports "The specified plugin cache dir /cache/plugins cannot be opened" (it goes on and installs anyway). The mount point always exists.

The parameter is `--variables`, not `--vars`: a parameter named `vars` would shadow the helper of `lib/common.lask` that drops the `null` entries.

`--variables` is how secrets reach Terraform: as `TF_VAR_*` variables of the environment, never on the command line. A secret value is masked if the caller bound it with `!!` — masking matches values, wherever they end up.

### 5.3 `ansible`

A recipe, since there is no official image: the Python image plus `ansible-core`, whose version is a build argument and so part of the recipe hash (10.3). It therefore takes no `--tag` (§4.4); a new version is a new release of this module.

| Parameter | Sets |
| --- | --- |
| `--config` | `ANSIBLE_CONFIG` (a path inside the project) |
| `--inventory` | `ANSIBLE_INVENTORY` |
| `--host_key_checking: Bool = true` | `ANSIBLE_HOST_KEY_CHECKING` |
| `--ssh_dir` | mounts `<dir>:/root/.ssh:ro` |
| `--vault_password_file` | mounts the file read-only at `/run/secrets/vault-password`; `ANSIBLE_VAULT_PASSWORD_FILE` points to it |

Defaults: `PYTHONUNBUFFERED=1`, so a playbook's output reaches the command log as it happens (12.3) rather than when the buffer fills.

### 5.4 `go`

| Parameter | Sets |
| --- | --- |
| `--goproxy` | `GOPROXY` |
| `--goprivate` | `GOPRIVATE` |
| `--goflags` | `GOFLAGS` |
| `--cgo_enabled` | `CGO_ENABLED` (`"0"` / `"1"`; a `String` so that "not given" exists) |
| `--goos`, `--goarch` | `GOOS`, `GOARCH` |
| `--cache_dir` | mounts at `/cache`; `GOMODCACHE=/cache/mod`, `GOCACHE=/cache/build` |

### 5.5 `node` (implemented)

| Parameter | Sets |
| --- | --- |
| `--tag` | the image, `node:<tag>`; default `24.21.0-alpine3.24`, an LTS release |
| `--node_env` | `NODE_ENV` |
| `--node_options` | `NODE_OPTIONS` |
| `--registry` | `npm_config_registry` |
| `--npm_token!!` | `NPM_TOKEN`, for an `.npmrc` that reads `${NPM_TOKEN}` |
| `--cache_dir` | mounts at `/cache`; `npm_config_cache=/cache/npm` |

Default: `npm_config_update_notifier=false`, so npm prints no update notice into the command log.

### 5.6 `python` (implemented)

| Parameter | Sets |
| --- | --- |
| `--tag` | the image, `python:<tag>`; default `3.12.14-alpine3.24` |
| `--index_url!!` | `PIP_INDEX_URL` — secret, since a private index URL often carries a credential |
| `--extra_index_url!!` | `PIP_EXTRA_INDEX_URL` |
| `--cache_dir` | mounts at `/cache`; `PIP_CACHE_DIR=/cache/pip` |

Defaults: `PYTHONUNBUFFERED=1` (as §5.3), `PYTHONDONTWRITEBYTECODE=1` — the project is bind-mounted and the container runs as root, so bytecode would leave root-owned `__pycache__` directories in the working tree — and `PIP_DISABLE_PIP_VERSION_CHECK=1`.

### 5.7 `java`, `maven`, `gradle`

| Function | Parameter | Sets |
| --- | --- | --- |
| all three | `--java_tool_options` | `JAVA_TOOL_OPTIONS` |
| `maven` | `--maven_opts` | `MAVEN_OPTS` |
| `maven` | `--cache_dir` | mounts at `/cache`; appends `-Dmaven.repo.local=/cache/m2` to `MAVEN_OPTS` |
| `gradle` | `--gradle_opts` | `GRADLE_OPTS` |
| `gradle` | `--cache_dir` | mounts at `/cache`; `GRADLE_USER_HOME=/cache/gradle` |

### 5.8 `cc`

| Parameter | Sets |
| --- | --- |
| `--cc`, `--cxx` | `CC`, `CXX` |
| `--cflags`, `--cxxflags`, `--ldflags` | `CFLAGS`, `CXXFLAGS`, `LDFLAGS` |
| `--makeflags` | `MAKEFLAGS` |

The official `gcc` image has no CMake. A CMake entry needs a recipe, which this module can ship beside it (lask#48).

## 6. Consumer Usage

```text
lask deps add tools --git https://github.com/lask-task-runner/lask-module-tools --rev <rev>
```

```lask
import * as tools from "tools"
import command { "go", "gofmt" } from "tools"        // toolchains: as they are

command { "aws" } on tools.aws(profile = "dev", config_dir = tools.home(".aws"))

test(): String = $ go test ./...
deploy(): String = $ aws s3 sync web/dist s3://my-bucket --delete

// Terraform with the same AWS account.
plan(): String = do {
  tf = tools.terraform(with = [tools.aws_setup(profile = "dev", config_dir = tools.home(".aws"))])
  $[tf] terraform -chdir=infra plan -input=false
}
```

## 7. Repository Layout

```text
lask-module-tools/
  LICENSE
  README.md             # conventions, the tools, copy-paste declarations
  main.lask             # the public API: `export { ... } from` lines only
  lib/
    common.lask         # ToolSetup, home, and what every tool is built from
    aws.lask            # one file per tool
  images/
    ansible/Dockerfile  # the Ansible recipe
  example/
    main.lask           # a project using the module
  test/
    selftest.lask
  doc/
    design.md
```

Each tool lives in `lib/<tool>.lask` and `main.lask` re-exports its public functions, as chapter 5 says a multi-file API is published. An import of a dependency reaches its `main.lask` and nothing else, so only what `main.lask` lists reaches a project. The helpers in `lib/common.lask` are public within the tree, where each tool's file imports them, and invisible outside it. A tool's exported command words are re-exported the same way, with `export command { ... } from`.

The command line of this repository reaches a re-exported function as it reaches a declared one: `lask eval aws --profile dev`, `lask run aws --help`.

## 8. Testing

1. **Static gate.** `lask check` on `main.lask`, `example/main.lask` and `test/selftest.lask`. (`lask envs` reports each tool's image as `<dynamic>`: its reference is built from `--tag`, §4.4.)
2. **Exact environments, without Docker.** An environment compares structurally, so `test/selftest.lask` states each expected environment in full and compares with `==`: the image `--tag` picks, which variables are set, that `null` leaves one out while `""` sets it, the precedence of §4.2, and the mounts. `lask eval --module test/selftest.lask all`.
3. **Declarability.** The selftest declares a command on each function, so a function that stops being effect-free fails `lask check` (ch. 5).
4. **Smoke, with Docker.** Each tool's version command, through its declared command word: `lask env build --module test/selftest.lask`, then `lask eval --module test/selftest.lask smoke`. The selftest writes the images it expects as literals, and a reference written as a literal resolves through the lock alone (lask spec 10.4), even where a tool computes the same reference at run time; `lask env build` pulls and pins them.

## 9. Adding a Tool

1. Pick the image by the rules of §5 and update the catalog table.
2. Write `<tool>(...)` in `lib/<tool>.lask` from what `lib/common.lask` provides, with `--tag` defaulting to the chosen release (§4.4), and `<tool>_setup` only if another tool needs its credentials (§4.3).
3. Re-export them from `main.lask`. If its default environment is useful as it is (§3), declare its command words in its file and re-export them too, with `export command { ... } from`.
4. Add its cases to `test/selftest.lask`, and its section to the README.

## 10. Prerequisites in Lask

Every gap in Lask found while designing this module has been closed:

| | Was | Resolved by |
| --- | --- | --- |
| P1 | A recipe in an imported module was read from the importing project's directory. | lask#48 |
| P2 | A namespace import could not reach a name its target re-exports. | lask#45 |
| P3 | A container option could not say "not given"; `""` reached the daemon. | lask#49, lask#50 |
| P4 | Registry images were not pinned in the lock, and runs pulled by tag. | lask#47 |
| P5 | The CLI could not invoke a function its module re-exports. | lask#46 |

Before those: a command declaration takes a call or a namespace member as its environment, and command words can be exported and imported (lask#44).

## 11. Open Questions

- **Root-owned files in the working tree.** Every image here runs as root and the project is bind-mounted, so anything a tool writes there is root-owned on a Linux host. Running as the host user takes `user` (§4.5) and the host's uid.
- **SSH keys in the Ansible container.** A read-only mount of `~/.ssh` into a root container may be refused by `ssh` for file ownership. An agent socket is the alternative, but its host path differs between Linux and Docker Desktop.
- **`home()` on Windows**, where `HOME` is usually unset and `USERPROFILE` is the equivalent.
- **v2 candidates.** `gcloud` and `az` (each with a `_setup`), `kubectl`, `helm`, `terragrunt`, and the Docker CLI, which needs the daemon socket mounted and widens the boundary of 10.7 further than anything in v1.
