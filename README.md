# lask-module-tools

Ready-made execution environments for the tools a [Lask](https://github.com/lask-task-runner/lask) task runs. Each tool is a function that returns an `Environment`; its keyword parameters are the release of its image, and what the tool needs from the environment it runs in — the variables it reads, its credentials, and the host directories it has to see.

```lask
import * as tools from "tools"

command { "aws" } on tools.aws(profile = "dev", config_dir = tools.home(".aws"))

deploy(): String = $ aws s3 sync web/dist s3://my-bucket --delete
```

The tools themselves are not wrapped: `$ aws s3 sync ...` is already the clearest way to say it. Design and the tools planned next: [doc/design.md](doc/design.md).

Requires a Lask whose container options take `null` (lask#49) and accept a typed list or table (lask#50), and whose secrets may be `null` (lask#51).

Pull a tool's image once before the first command that uses it, e.g. `docker pull amazon/aws-cli:2.36.41`. A tool builds its image reference from `--tag` when it runs, so Lask treats the reference as computed at run time (lask spec 10.3): `lask env build` cannot pull or pin it, and a run never pulls.

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

## Layout

Each tool lives in `lib/<tool>.lask`, built from `lib/common.lask`, and `main.lask` re-exports its public functions. Only what `main.lask` lists reaches a project, and the command line of this repository reaches each of them as well: `lask eval aws --profile dev`, `lask run aws --help`.

## Testing

```text
lask check                                       # the module
lask check --module example/main.lask            # a project using it
lask eval --module test/selftest.lask all        # every environment, compared exactly; no Docker
lask env build --module test/selftest.lask       # pulls and pins the images the selftest names
lask eval --module test/selftest.lask smoke      # runs each tool; needs Docker
```
