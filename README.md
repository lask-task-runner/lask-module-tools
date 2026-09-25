# lask-module-tools

Ready-made execution environments for the tools a [Lask](https://github.com/lask-task-runner/lask)
task runs. Each tool is a function that returns an `Environment`; its keyword
parameters are the release of its image, and what the tool needs from the
environment it runs in — the variables it reads, its credentials, and the host
directories it has to see.

```lask
import * as tools from "tools"

command { "aws" } on tools.aws(profile = "dev", config_dir = tools.home(".aws"))

deploy(): String = $ aws s3 sync web/dist s3://my-bucket --delete
```

The tools themselves are not wrapped: `$ aws s3 sync ...` is already the clearest
way to say it. Design and the tools planned next: [doc/design.md](doc/design.md).

Requires a Lask whose container options take `null` (lask#49) and accept a typed
list or table (lask#50), and whose secrets may be `null` (lask#51).

Pull a tool's image once before the first command that uses it, e.g.
`docker pull amazon/aws-cli:2.36.41`. A tool builds its image reference from `--tag`
when it runs, so Lask treats the reference as computed at run time (lask spec 10.3):
`lask env build` cannot pull or pin it, and a run never pulls.

## Install

```text
lask deps add tools --git https://github.com/lask-task-runner/lask-module-tools --rev <rev>
```

## Conventions

Every tool follows the same rules:

- **`--tag` picks the release of the tool's image**, and defaults to the one this
  module was written and tested against. Pull the image for the tag you run (above).
- **A parameter left `null` sets nothing.** Every parameter defaults to `null`, and
  a variable given `null` is left out. `""` is a value: it sets its variable to the
  empty string.
- **Secrets are `!!` parameters**, so they are masked in the command log, including
  in the environment it records — whatever the caller passed. They default to `null`
  like the rest, and a `null` secret registers nothing for masking.
- **A host path is mounted, never expanded.** Lask starts `docker` without a shell, so
  `~` would reach the daemon as a directory named `~`. Use `tools.home(".aws")`.
- **`--with: Array<ToolSetup>`** carries another tool's variables and mounts, so the
  Terraform AWS provider can run with the same account as the AWS CLI:
  `terraform(with = [tools.aws_setup(profile = "dev")])`.
- **`--extra_env: Map<String>`** sets any variable the tool has no parameter for.
- **Precedence**, later wins: the tool's defaults, `--extra_env`, each setup in
  `--with`, the tool's own named parameters.

## Tools

### `aws` — AWS CLI v2

Image `amazon/aws-cli:<tag>`, `2.36.41` unless `--tag` says otherwise. Declare the
command word in your module:

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

`aws_setup(...)` takes the same parameters except `--with` and `--extra_env`, and
returns them as a `ToolSetup` for another tool.

**Locally**, log in on the host (`aws sso login`) and pass
`config_dir = tools.home(".aws")`. The mount is read-only, so an expired token fails
instead of being refreshed; log in again on the host. A container cannot usefully run
the login itself: it cannot open the host's browser, and its token cache would be
discarded with it.

**In CI**, pass the credentials. A container sees no variable of the host unless it is
given one; `find_env` gives `null` for a variable that is not set:

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

The module exports no `aws` command word. An AWS CLI with no profile and no
credentials would fail at the first call, so the declaration belongs in the project
that knows which account to use. Declare it once and `export` it from your own
module if several of your modules need it.

More in [example/main.lask](example/main.lask).

## Layout

Each tool lives in `lib/<tool>.lask`, built from `lib/common.lask`, and
`main.lask` re-exports its public functions. Only what `main.lask` lists reaches a
project, and the command line of this repository reaches each of them as well:
`lask eval aws --profile dev`, `lask run aws --help`.

## Testing

```text
lask check                                       # the module
lask check --module example/main.lask            # a project using it
lask eval --module test/selftest.lask all        # every environment, compared exactly; no Docker
lask eval --module test/selftest.lask smoke      # runs the CLI; needs Docker and
                                                 #   amazon/aws-cli:2.36.41 pulled
```
