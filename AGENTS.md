# Taskfiles

This repository holds reusable [Task](https://taskfile.dev) Taskfiles that other projects include, usually as a Git submodule at `.taskfiles`. Each top-level YAML file wraps one tool, such as Terraform or Docker. Its public API is the uppercase variables a consumer sets and the tasks the consumer calls. The consumer's `Taskfile.yml` supplies project values, and these files supply the commands.

There is no build, test suite, or CI. To check a change, include the edited file in a scratch project and run its tasks.

## Each file stands alone

Every Taskfile is independent. It does not include another Taskfile from this repository, and it does not declare root-level `vars` or `env`. A consumer can include any one file, under any namespace, more than once.

## Variables are the public API

Treat a change to a public variable or task name as a breaking change for every consumer.

- Public variables are uppercase and start with the tool's prefix. Lowercase variables are internal.
- The prefix must not collide with names the wrapped tool reads from the environment. That is why Terraform variables start with `TERRAFORM_` and not `TF_`, and why Docker uses `DOCKER_BUILD_CONTEXT` and not `DOCKER_CONTEXT`.
- Each tool-specific variable that a task needs has a default in the task's own `vars`. Variables without a sensible default go under `requires`.

## Defaults live in task-level vars

Declare defaults as `'{{.NAME | default "value"}}'` in each task's `vars`, shared across the file with a YAML anchor. A file-level `vars` entry in an included Taskfile overwrites the consumer's root variable of the same name, and variables passed on a task call never reach its template. Task-level `vars` avoid both problems, so the consumer's value always wins over the default.

Dependencies do not inherit call variables. A task that calls another task through `deps` or `cmds` forwards every variable the callee needs.

## Bad input fails before any command runs

A missing or invalid value stops the task with a clear message, so a typo cannot reach the wrapped tool. Use `requires` for missing variables and for boolean values, which accept only `true` or `false`. Use `preconditions` for values that `requires` accepts but that are wrong when empty. A mistyped directory variable must fail, not create the directory, so tasks change directory with `cd {{q .DIR}} &&` and never use the `dir` key.

## Values reach the shell quoted

Values that hold one item pass through the `q` template function. Values that hold many items are space-separated strings, parsed with `splitArgs`, and each item passes through `q`. This keeps spaces and shell characters in a consumer's value from splitting or running as commands.

The exceptions are the `*_ARGS`, `*_FLAGS`, and `*_ENV` variables. They are deliberate escape hatches, and their values are shell text.
