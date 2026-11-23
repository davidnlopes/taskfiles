# Taskfiles

Reusable [Task](https://taskfile.dev) Taskfiles that you include in your own repositories. Each file covers one tool, such as Terraform or Docker, and exposes a public API: variables you set and tasks you call. Your project supplies the values, and these files supply the commands.

| Taskfile | Runs |
| --- | --- |
| `terraform.yml` | Terraform or OpenTofu against one root module |
| `golang.yml` | Format, lint, test, and build for one Go module |
| `node.yml` | Install and package scripts for one npm package |
| `docker.yml` | Image build and push as separate tasks |

To list the tasks of an included Taskfile, run `task --list`. The `vars` block of the first task in each file holds the optional variables and their defaults. Required variables appear under `requires`.

## Requirements

- Task 3.47 or later.
- A root `Taskfile.yml` with `version: '3'`. Task rejects an include whose `version` differs from the root.
- The tool that each Taskfile wraps. Terraform tasks need Terraform 1.4 or later, and Docker tasks need Docker 23.0 or later for BuildKit.

## Add the repository to a project

To add this repository as a Git submodule at `.taskfiles`, run:

```bash
git submodule add https://github.com/davidnlopes/taskfiles.git .taskfiles
```

Clone projects that use it with `git clone --recurse-submodules`. In CI, enable recursive submodule checkout.

To update to the latest commit, run:

```bash
git submodule update --remote .taskfiles
```

## Use a Taskfile

Include the file in your `Taskfile.yml`, set the variables you need, and call the tasks:

```yaml
version: '3'

vars:
  TERRAFORM_DIR: '{{.TERRAFORM_DIR | default "terraform"}}'
  TERRAFORM_WORKSPACE: '{{.TERRAFORM_WORKSPACE | default "my-project-dev"}}'

includes:
  tf: .taskfiles/terraform.yml
```

```bash
task --list
task tf:plan TERRAFORM_VAR_FILES=configs/dev.tfvars
task tf:apply TERRAFORM_AUTO_APPROVE=true
```

You choose each namespace in `includes`. This README uses `tf`, `go`, and `docker`.

### Conventions

These rules hold for every Taskfile:

- Public variables are uppercase and start with the tool's prefix, for example `TERRAFORM_DIR`. Lowercase variables are internal and can change without notice.
- Optional variables have a default. Required variables have none. A task stops before it runs anything if a required variable is missing, and before its own commands if the value is empty.
- Boolean variables accept only `true` or `false`. Any other value stops the task with an error.
- Variables that hold many values are space-separated strings. To keep a space inside a value, quote it, for example `DOCKER_BUILD_ARGS='TITLE="two words" VERSION=1'`. YAML lists are not supported.
- Tasks that run commands also accept an `*_ARGS` or `*_FLAGS` variable for flags that the Taskfile does not expose.
- Prefixes avoid names that the wrapped tool reads from the environment. For example, Terraform variables start with `TERRAFORM_`, not `TF_`, because Terraform reads `TF_*` itself.

### Set variable values

Write each value in your Taskfile as `'{{.NAME | default "value"}}'`, in root `vars` or in the `vars` of an include. The command line and the environment can then override it.

Values resolve in this order, highest first:

1. `vars` on a `task:` call
2. The command line
3. The environment
4. The value in your Taskfile
5. The default in this repository

A plain `NAME: value` behaves differently. In the `vars` of an include, it beats the command line and the environment. In root `vars`, it beats the environment. It also beats the command line for any include that uses the `taskfile:` key.

An empty value on the command line falls back to the value in your Taskfile when you use the `default` form.

### Include a Taskfile more than once

To run the same Taskfile against several modules, include it under several namespaces:

```yaml
includes:
  api:
    taskfile: .taskfiles/golang.yml
    vars:
      GO_DIR: services/api
  worker:
    taskfile: .taskfiles/golang.yml
    vars:
      GO_DIR: services/worker
```

### Wrap a task

To run your own step before a shared task, or to give it a project-specific name, define a task that calls it. Add a `desc` to list your task in `task --list`:

```yaml
tasks:
  deploy:
    desc: Deploy the infrastructure
    deps: [check-credentials]
    cmds:
      - task: tf:apply

  check-credentials:
    cmds:
      - aws sts get-caller-identity
```

## Notes per Taskfile

### Terraform

To use OpenTofu, set `TERRAFORM_BIN=tofu`.

Terraform runs with `-chdir=$TERRAFORM_DIR`, so it resolves relative paths in `TERRAFORM_VAR_FILES` and `TERRAFORM_*_ARGS` against `TERRAFORM_DIR`.

To give Terraform an environment variable that your other tasks must not see, use `TERRAFORM_ENV`. A root `env` in your Taskfile reaches every task, and an `env` on a task that calls a Terraform task does not reach it. The value is shell text, so quote any value that contains a space, for example `NAME="two words"`.

### Go

`GO_BUILD_OS`, `GO_BUILD_ARCH`, and `GO_BUILD_CGO_ENABLED` apply only to `build`. If you export `GOOS` instead, `go test` and `go vet` also compile for that OS, and `go test` cannot run a binary built for another OS.

Commands run from `GO_DIR`, so Go resolves relative paths in `GO_BUILD_OUTPUT`, `GO_COVER_PROFILE`, and `GO_COVER_HTML` against `GO_DIR`.

To give Go an environment variable that your other tasks must not see, use `GO_ENV`. It follows the same quoting rule as `TERRAFORM_ENV`, and `cover-html` forwards it to `test`.

To run tests through another runner, set `GO_TEST_CMD`. For example, [gotestsum](https://github.com/gotestyourself/gotestsum) prints each test as it finishes, while `go test` holds back a package's output until the whole package is done. End the command with `--`, so that gotestsum passes the flags and packages after it to `go test`:

```yaml
includes:
  go:
    taskfile: .taskfiles/golang.yml
    vars:
      GO_TEST_CMD: '{{.GO_TEST_CMD | default "go tool gotestsum --format testname --"}}'
```

To build a second binary, call `build` again with other values:

```yaml
tasks:
  build-seed:
    cmds:
      - task: go:build
        vars:
          GO_BUILD_PKG: ./cmd/seed
          GO_BUILD_OUTPUT: '{{.ROOT_DIR}}/bin/seed'
```

### Node

Every task runs a script from `package.json`, so `package.json` stays the source of truth for the commands. Only npm is supported.

`install` runs once per `task` invocation and skips when `node_modules/.package-lock.json` is newer than `package-lock.json`. Other tasks do not call `install`. To install first, add a `deps` entry that calls `install` in your own task.

`lint` is check-only when your scripts are. Point `NODE_LINT_SCRIPTS` at scripts that do not rewrite files.

### Docker

Run `docker login` before you push.

`DOCKER_IMAGE` is required. `DOCKER_REGISTRY` is required by `push`. To test an image and then push the same image, run `build` and then `push` with the same `DOCKER_REGISTRY`, `DOCKER_IMAGE`, and `DOCKER_TAGS`:

```yaml
vars:
  DOCKER_REGISTRY: '{{.DOCKER_REGISTRY | default "ghcr.io/my-org"}}'
  DOCKER_IMAGE: '{{.DOCKER_IMAGE | default "my-app"}}'

includes:
  docker: .taskfiles/docker.yml
```

```bash
task docker:build DOCKER_TAGS="1.2.0 latest"
task docker:push DOCKER_TAGS="1.2.0 latest"
```

To publish a multi-platform image, pass `--push` to `build`, because a local image store might not hold one. The builder must support every platform you list. To select a builder, pass `--builder` in `DOCKER_BUILD_FLAGS`:

```bash
task docker:build DOCKER_TAGS=1.2.0 DOCKER_BUILD_PLATFORMS="linux/amd64 linux/arm64" DOCKER_BUILD_FLAGS=--push
```

The build context variable is `DOCKER_BUILD_CONTEXT`, because the Docker CLI reads `DOCKER_CONTEXT` to select a Docker context.

## Develop this repository

To check a change, include the edited file in a scratch project and run its tasks with `task`. The repository has no test suite or CI of its own.

## License

See [`LICENSE`](LICENSE).
