# Docker, CI and repo config

Read this when you write or change a Dockerfile, a CI workflow, a build or install script,
or repo-wide config (linter, formatter, workspace, deploy). The rules in
[SKILL.md](SKILL.md) still apply. The goal: adding the next service, package or file needs
no edit here.

## Name no instance

- **Match by convention, not by list.** Config points at folders and globs, never at each
  service, package or file. `"files": ["services/**", "packages/**"]`, not one entry per
  service. `COPY packages ./packages`, not one `COPY` per package.
- **One folder per kind of thing.** Shared libraries in `packages/`, deployable services in
  `services/`, so a glob can find each kind. A new item goes in its folder and is picked up.
- **One install path for all packages.** Use the package manager's workspaces, or one
  script that loops over the folder. Never a line per package in a Dockerfile, workflow or
  script.
- **A list that must exist is generated.** When a tool really needs explicit names (a CI
  matrix, a build graph), derive them from the folders in one script or step. Don't keep a
  second hand-written copy.
- **No paths or names repeated across files.** A path, image name, version or port used in
  more than one file gets one owner: an env file, a build arg or a shared config. The other
  files read it.

## Dockerfiles

- **Multi-stage.** Build in one stage, and copy only the built output and the runtime
  dependencies into a small final stage.
- **Cache-friendly order.** Copy the dependency manifests and lockfile first, run a frozen
  install, then copy the source. Source edits then skip the install layer. In a monorepo,
  prune to the target's workspace first (`turbo prune --docker` or the package manager's
  equivalent).
- **Exclude with `.dockerignore`**, not by copying files one at a time. Exclude `.git`,
  `node_modules`, build output and `.env*`.
- **Pin the base image** to a version tag. Run as a non-root user.
- **No secrets in images.** Pass them at runtime or as build secrets, never as `ARG`, `ENV`
  or a copied file.

## CI workflows

- **Shared steps live in one place.** Steps repeated across jobs become one composite
  action. A job repeated across workflows becomes one reusable workflow that takes inputs.
- **CI runs the project's own scripts.** A job calls `bun run test` or `make lint`, the
  same command a developer runs. No long inline shell that exists only in CI.
- **Per-service jobs come from a matrix** built from the folders, with path filters as
  globs. Not one copied workflow per service.
- **Secrets come from the secret store**, scoped to the job that needs them.

## Self-check

Reread your diff and fix:

- every line that names one service, package or file where a folder or glob would match
- every step, job or path copied from another workflow or Dockerfile
- every `COPY` or install line that a new package would have to repeat
