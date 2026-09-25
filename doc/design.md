# lask-module-tools Design Document

`lask-module-tools` is a catalog of execution environments for the tools a Lask
task most often runs: cloud CLIs, infrastructure tools and language toolchains.
Each entry pins an image and exposes, as keyword arguments, what a caller has to
supply for that tool to work — its environment variables, its credentials, and
the host directories it needs to see.

It replaces `lask-aws` and `lask-terraform`, which are to be retired. Their typed
functions are not carried over.

Section numbers refer to the Lask specification (`lask/doc/spec.md`).

## 1. Goals

- One place where the image for each tool is chosen and pinned, so a project
  never has to find a working image or tag on its own.
- Every environment is configured by named arguments, not by knowing which
  variables a tool reads: `aws(profile = "dev")`, not
  `env = {"AWS_PROFILE": "dev"}`.
- Environments compose. A tool that needs another tool's credentials — the
  Terraform AWS provider, an Ansible dynamic inventory — takes them as a value,
  without the caller restating them.

## 2. Non-Goals

- No wrapper functions around the tools themselves. `$ aws s3 sync ...` and
  `$ go test ./...` are already the clearest way to say it, and a wrapper per
  subcommand is an API to keep in step with every release of the tool.
- No credential provisioning. The module forwards what it is given and never
  runs a login flow. For SSO, log in on the host and mount the resulting
  directory (§5.1): a container cannot do it usefully, because it cannot open
  the host's browser and its token cache is discarded with it.
- No container options beyond variables and mounts in v1 (§4.5).

## 3. One Function per Tool

Every tool is a function returning an `Environment`:

```lask
aws(--profile: String = "", ...): Environment = #docker("amazon/aws-cli:2.36.41", ...)
```

It serves both ways of running a command, because the environment of a command
declaration is an ordinary expression (ch. 5, since lask#44):

```lask
command { "aws" } on tools.aws(profile = "dev")        // then `$ aws s3 ls`
check(): String = $[tools.aws(profile = "ci")] aws sts get-caller-identity
```

The declaration's environment is evaluated when a command runs, and may not have
an effect. Every function here satisfies that: it builds a value from its
arguments and from `get_env`, and runs nothing.

### Command words

A module can export command words, and a project imports them by name:

```lask
import command { "go", "gofmt" } from "tools"
```

A tool **exports its command words when its default environment is useful as
it is**: a language toolchain usually needs nothing, so `go`, `node` and the
rest export theirs, declared on the function called with no arguments.

A tool **does not export them when it needs configuring first**. `aws` with no
profile and no credentials fails at the first call, so the declaration belongs
to the project that knows which account to use. A project that needs it in
several modules declares it once and exports it from its own module.

## 4. Function Conventions

### 4.1 Parameters

Every parameter is a keyword parameter with a default, so every function can be
called with no arguments and every argument can be given from the CLI (11.2).

| Kind | Type | Default | Meaning of the default |
| --- | --- | --- | --- |
| Variable the tool reads | `String \| Null` | `null` | Not set. The variable is left out; `""` sets it to the empty string. |
| Secret | `String`, marked `!!` | `""` | Not set. A `!!` binding has to be a `String` (6.10), so a secret cannot be `null`, and `""` is its "not given". The value is masked in the command log, including in the environment metadata, whatever the caller passed (12.3). |
| Host directory to read | `String \| Null` (a host path) | `null` | Not mounted. Mounted read-only when given. |
| Cache directory | `--cache_dir: String` | `""` | No cache. When given, mounted read-write at `/cache`, and the tool's cache variables point below it. |
| Tool-specific table | `Map<String>` | `{}` | Nothing. |
| Setups from other tools | `--with: Array<ToolSetup>` | `[]` | Nothing (§4.3). |
| Anything else | `--extra_env: Map<String>` | `{}` | Nothing. The escape hatch for a variable this module has no parameter for. |

`null` meaning "not given" is the convention of the whole module: a parameter
that was not given is `null`, and the internal `vars` helper drops the `null`
entries of a variable map. `""` is a value a caller may mean — `AWS_PAGER=""`
is how the pager is turned off — so it is passed as given. Secrets are the one
exception above; `unset_if_empty` turns their `""` into the `null` the others
use.

A path is always a host path, and is never expanded: Lask starts `docker`
without a shell, so `~` reaches the daemon as a directory named `~`. The
exported `home(path)` builds one under the invoking user's home directory.

### 4.2 Precedence

Variables are layered, the later layer winning on a name present in both:

1. The function's own defaults (e.g. `AWS_PAGER=""`)
2. `--extra_env`
3. Each `ToolSetup` in `--with`, left to right
4. The function's own named parameters

Named parameters win because they are the most specific thing the caller said.
`--extra_env` sits below `--with` so that it can override a default but cannot
silently undo a credential a setup supplies. Mounts are concatenated in the
same order.

### 4.3 Composition: `ToolSetup`

```lask
type ToolSetup = Record<env: Map<String>, volumes: Array<String>>
```

A tool whose credentials other tools need publishes a second form,
`<tool>_setup(...)`, taking the same parameters as its function and returning
the variables and mounts instead of an environment. The function is defined over
its own setup, so the two cannot drift. A setup carries none of its tool's
defaults: `AWS_PAGER` concerns the AWS CLI, not the Terraform provider.

```lask
creds = tools.aws_setup(profile = "dev", config_dir = tools.home(".aws"))

// The Terraform image, with the AWS provider's credentials.
tf = tools.terraform(vars = {"app_version": "1.2.3"}, with = [creds])
```

The same `creds` value can be handed to `ansible(with = [...])` for an AWS
dynamic inventory, and a future `gcloud_setup` slots into the same list.

### 4.4 The Image Is Not a Parameter

The image tag is a literal inside each function, never built from an argument:

```lask
go(--version: String = "1.25"): Environment = #docker("golang:#{version}")
// lask envs reports: <dynamic> (docker: <dynamic image>)
```

A dynamic image cannot be enumerated, pinned, or materialized ahead of time
(10.3, 11.4), which would undo the point of the catalog. The version moves with
the rev of this module, which the consumer's lock file pins.

If selectable versions are added later, they are selected among literals, which
keeps every one of them enumerable, and an unknown version fails:

```lask
go(--version: String = "1.25"): Environment = do {
  if (version == "1.25") { return #golang:1.25 }
  if (version == "1.24") { return #golang:1.24 }
  fail(error(2, "go: unsupported version '#{version}' (1.25, 1.24)"))
}
```

The cost is that enumeration over-approximates (11.4): every task that reaches
`go()` is reported as needing both images. So at most two versions per tool, and
not in v1.

### 4.5 Container Options

A container option given `null` is left out (10.2), so a tool can take
`--user: String | Null = null` and pass it straight to `#docker(...)`. v1's
tools expose variables and mounts only, since none of them needs more; a tool
that does — a `platform` for an image published for one architecture, a `user`
for §11's root-owned files — adds the parameter and passes it through.

## 5. Catalog

| Function | Image | Exported command words | Status |
| --- | --- | --- | --- |
| `aws` | `amazon/aws-cli:2.36.41` | none (§3) | **implemented** |
| `terraform` | `hashicorp/terraform:1.16.2` | none — it needs provider credentials | planned |
| `ansible` | recipe `images/ansible/Dockerfile` | none — it needs an inventory | planned |
| `go` | `golang:<pin>` | `go`, `gofmt` | planned |
| `node` | `node:<pin>-alpine<pin>` | `node`, `npm`, `npx`, `corepack` | planned |
| `python` | `python:3.12.14-alpine3.24` | `python`, `python3`, `pip`, `pip3` | planned |
| `java` | `eclipse-temurin:<21 pin>-jdk` | `java`, `javac`, `jar` | planned |
| `maven` | `maven:<3.9 pin>-eclipse-temurin-21` | `mvn` | planned |
| `gradle` | `gradle:<8 pin>-jdk21` | `gradle` | planned |
| `cc` | `gcc:<pin>` | `gcc`, `g++`, `cc`, `c++`, `make` | planned |

`<pin>` is chosen when the entry is implemented, with these rules:

- An exact tag, down to the patch release and the base-image suffix. The lock
  pins the digest a tag resolved to (lask#47), so a project does not move when
  the tag does; the exact tag is what says, to a reader, which release it is.
- A release still in support. In particular, not `node:20.20.2-alpine3.23`,
  which the `lask` examples still use: Node 20 reached end of life on
  2026-04-30.
- The Ansible recipe builds from the same Python pin as `python`.

### 5.1 `aws` / `aws_setup` (implemented)

| Parameter | Sets |
| --- | --- |
| `--profile` | `AWS_PROFILE` |
| `--region` | `AWS_REGION` and `AWS_DEFAULT_REGION` — the CLI and the SDKs read different ones |
| `--endpoint_url` | `AWS_ENDPOINT_URL` — LocalStack or another AWS-compatible endpoint |
| `--access_key_id` | `AWS_ACCESS_KEY_ID` |
| `--secret_access_key!!` | `AWS_SECRET_ACCESS_KEY` |
| `--session_token!!` | `AWS_SESSION_TOKEN` |
| `--config_dir` | mounts `<dir>:/root/.aws:ro` |

Default: `AWS_PAGER=""`. The CLI would otherwise hold output behind `less`, which
blocks under `lask cmd` on a terminal.

Local use is `config_dir = tools.home(".aws")` after `aws sso login` on the host.
The mount is read-only, so an expired token fails rather than being refreshed:
the fix is to log in again on the host. CI passes the three credentials.

The access key id is not a `!!` parameter: it identifies a key and is not a
secret, and masking it would hide from the log which key a run used.

### 5.2 `terraform`

| Parameter | Sets |
| --- | --- |
| `--workspace` | `TF_WORKSPACE` |
| `--log` | `TF_LOG` |
| `--vars: Map<String>` | `TF_VAR_<name>` for each entry |
| `--cache_dir` | mounts at `/cache`; `TF_PLUGIN_CACHE_DIR=/cache/plugins` |

Defaults: `TF_IN_AUTOMATION=1`, `TF_INPUT=0`, so nothing ever waits on a prompt.
Provider credentials come in through `--with`.

`--vars` is how secrets reach Terraform: as `TF_VAR_*` variables of the
environment, never on the command line. A secret value is masked if the caller
bound it with `!!` — masking matches values, wherever they end up.

### 5.3 `ansible`

A recipe, since there is no official image: the Python pin plus `ansible-core`,
whose version is a build argument and so part of the recipe hash (10.3).

| Parameter | Sets |
| --- | --- |
| `--config` | `ANSIBLE_CONFIG` (a path inside the project) |
| `--inventory` | `ANSIBLE_INVENTORY` |
| `--host_key_checking: Bool = true` | `ANSIBLE_HOST_KEY_CHECKING` |
| `--ssh_dir` | mounts `<dir>:/root/.ssh:ro` |
| `--vault_password_file` | mounts the file read-only at `/run/secrets/vault-password`; `ANSIBLE_VAULT_PASSWORD_FILE` points to it |

Defaults: `PYTHONUNBUFFERED=1`, so a playbook's output reaches the command log as
it happens (12.3) rather than when the buffer fills.

### 5.4 `go`

| Parameter | Sets |
| --- | --- |
| `--goproxy` | `GOPROXY` |
| `--goprivate` | `GOPRIVATE` |
| `--goflags` | `GOFLAGS` |
| `--cgo_enabled` | `CGO_ENABLED` (`"0"` / `"1"`; a `String` so that "not given" exists) |
| `--goos`, `--goarch` | `GOOS`, `GOARCH` |
| `--cache_dir` | mounts at `/cache`; `GOMODCACHE=/cache/mod`, `GOCACHE=/cache/build` |

### 5.5 `node`

| Parameter | Sets |
| --- | --- |
| `--node_env` | `NODE_ENV` |
| `--node_options` | `NODE_OPTIONS` |
| `--registry` | `npm_config_registry` |
| `--npm_token!!` | `NPM_TOKEN`, for an `.npmrc` that reads `${NPM_TOKEN}` |
| `--cache_dir` | mounts at `/cache`; `npm_config_cache=/cache/npm` |

### 5.6 `python`

| Parameter | Sets |
| --- | --- |
| `--index_url!!` | `PIP_INDEX_URL` — secret, since a private index URL often carries a credential |
| `--extra_index_url!!` | `PIP_EXTRA_INDEX_URL` |
| `--cache_dir` | mounts at `/cache`; `PIP_CACHE_DIR=/cache/pip` |

Defaults: `PYTHONUNBUFFERED=1` (as §5.3), `PYTHONDONTWRITEBYTECODE=1` — the
project is bind-mounted and the container runs as root, so bytecode would leave
root-owned `__pycache__` directories in the working tree — and
`PIP_DISABLE_PIP_VERSION_CHECK=1`.

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

The official `gcc` image has no CMake. A CMake entry needs a recipe, which this
module can ship beside it (lask#48).

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

Each tool lives in `lib/<tool>.lask` and `main.lask` re-exports its public
functions, as chapter 5 says a multi-file API is published. An import of a
dependency reaches its `main.lask` and nothing else, so only what `main.lask`
lists reaches a project. The helpers in `lib/common.lask` are public within the
tree, where each tool's file imports them, and invisible outside it. A tool's
exported command words are re-exported the same way, with
`export command { ... } from`.

The command line of this repository reaches a re-exported function as it
reaches a declared one: `lask eval aws --profile dev`, `lask run aws --help`.

## 8. Testing

1. **Static gate.** `lask check` on `main.lask`, `example/main.lask` and
   `test/selftest.lask`, and `lask envs` on `example/` to confirm every image is
   enumerated concretely, never as `<dynamic>`.
2. **Exact environments, without Docker.** An environment compares structurally,
   so `test/selftest.lask` states each expected environment in full and compares
   with `==`: which variables are set, that `""` leaves one out, the precedence
   of §4.2, and the mounts. `lask eval --module test/selftest.lask all`.
3. **Declarability.** The selftest declares a command on each function, so a
   function that stops being effect-free fails `lask check` (ch. 5).
4. **Smoke, with Docker.** Each tool's version command, through its declared
   command word: `lask eval --module test/selftest.lask smoke`.

## 9. Adding a Tool

1. Pick the image by the rules of §5 and update the catalog table.
2. Write `<tool>(...)` in `lib/<tool>.lask` from what `lib/common.lask` provides,
   and `<tool>_setup` only if another tool needs its credentials (§4.3).
3. Re-export them from `main.lask`. If its default environment is useful as it
   is (§3), declare its command words in its file and re-export them too, with
   `export command { ... } from`.
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

Before those: a command declaration takes a call or a namespace member as its
environment, and command words can be exported and imported (lask#44).

## 11. Open Questions

- **Root-owned files in the working tree.** Every image here runs as root and
  the project is bind-mounted, so anything a tool writes there is root-owned on
  a Linux host. Running as the host user takes `user` (§4.5) and the host's uid.
- **SSH keys in the Ansible container.** A read-only mount of `~/.ssh` into a
  root container may be refused by `ssh` for file ownership. An agent socket is
  the alternative, but its host path differs between Linux and Docker Desktop.
- **`home()` on Windows**, where `HOME` is usually unset and `USERPROFILE` is
  the equivalent.
- **v2 candidates.** `gcloud` and `az` (each with a `_setup`), `kubectl`,
  `helm`, `terragrunt`, and the Docker CLI, which needs the daemon socket
  mounted and widens the boundary of 10.7 further than anything in v1.
