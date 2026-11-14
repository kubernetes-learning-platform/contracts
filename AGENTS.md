# AGENTS.md: contracts

Instructions for AI coding agents working in this repository. `CLAUDE.md` is a symlink to this file, so Claude Code reads the same instructions.

## What this repo is

The Protobuf schema for kubernetes-learning-platform, a live Kubernetes lab with an optional AI helper. Every message that crosses a process boundary is defined here, once:

| Boundary | Transport | File |
|---|---|---|
| browser ↔ `api` | WebSocket, binary frames | `session.proto` |
| browser → `api` | REST | `rest.proto` |
| `api` ↔ `envmgr` | gRPC on the course VM | `envmgr.proto` |
| `api` → `envmgr` | RabbitMQ, sandbox start and teardown only | `jobs.proto` |
| `envmgr` ↔ `harness` | WebSocket, one `HarnessFrame` per message | `harness.proto` |

The repo has no runtime code. It publishes two packages from the same schema: `KubernetesLearning.Contracts` (NuGet, for `sandbox` and `api`) and `@kubernetes-learning-platform/contracts` (npm, for `web`).

- **Owner:** Filip. Teammates open schema PRs; Filip reviews them.
- **Status:** planned layout. contracts#1 sets up buf and the packages, contracts#2 the publishing.
- **Design:** `README.md` here, and ARCHITECTURE §5 in the `docs` repo. Where they disagree, the `docs` repo wins.

## Layout

```
contracts/
├── proto/kubernetes_learning/v1/          # common, envmgr, harness, topology, view, session, rest, jobs
├── buf.yaml                               # lint + breaking rules
├── buf.gen.yaml                           # protobuf-es codegen
├── dotnet/KubernetesLearning.Contracts/   # NuGet; Grpc.Tools compiles ../../proto at build time
├── ts/                                    # npm package; src/gen/ is generated, never committed
└── .github/workflows/                     # call platform's reusable workflows
```

## Build and check

Not set up yet; contracts#1 adds the exact commands here. The checks CI will run:

- `buf lint`
- `buf breaking --against '.git#branch=main'`
- build the .NET package and the TypeScript package

Run them before saying a change is done.

## Schema rules

- **Wire compatibility.** Never change a field's number or type, or reuse a number. Removing a field or enum value needs a `reserved` entry for both the number and the name.
- **Style.** Messages and enums in PascalCase, fields in snake_case. The first enum value is `<ENUM>_UNSPECIFIED = 0`, and every value is prefixed with the enum name.
- **Placement.** A message goes in the file for its boundary. `ClientMessage`, `ServerMessage` and `HarnessFrame` grow through their `oneof`.
- **Comments.** Every message and every non-obvious field gets a short comment; consumers read them as the API docs.
- **Generated code** (`ts/src/gen/`, `bin/`, `obj/`) is never committed.
- **Naming.** The proto package stays `kubernetes_learning.v1`. The name is a placeholder that is renamed before 1.0, so don't hardcode it anywhere new.

## Versioning

The PR title decides the release, so pick its type deliberately:

| PR title | Result while on `0.x` |
|---|---|
| `feat!:` or `fix!:` (breaking) | Minor release, e.g. `0.3.4` → `0.4.0` |
| `feat:` or `fix:` | Patch release, e.g. `0.3.4` → `0.3.5` |
| `docs:`, `ci:`, `chore:` and other types | Nothing is published |

- A PR fails if `buf breaking` finds a break and its title has no `!`.
- CI publishes both packages and pushes the `vX.Y.Z` tag. Never create tags or publish by hand.
- After publishing, a bot moves `sandbox`, `api` and `web` to the new version. Nobody opens a bump PR.

## Commits, branches and PRs

- **Always ask the user before creating a branch, committing or pushing.** Show the commit message when you ask.
- **Commits:** Conventional Commits only, one line: `type: short description`, lowercase, no trailing period. No body. No `Co-Authored-By` or any other attribution.
- **Types:** `feat`, `fix`, `docs`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`, `revert`, `style`. Add `!` for a breaking change.
- **Branches:** use the name in the `Branch:` line of the issue being worked on. Don't make one up. `gh issue develop <number> --name <branch> --checkout` creates it linked to the issue.
- **Where changes land:** while the repo is still empty, documentation goes straight to `main`. From the setup issues on (contracts#1), every change goes through a PR from the issue's branch.
- **PRs:** the title follows the commit format, at most 72 characters; it becomes the squash commit on `main`. The description is one or two plain sentences, plus `Closes #<number>`. Only Filip merges.

## Keep docs in sync

When a change touches this repo's design, layout, commands or conventions, update `AGENTS.md`, `README.md` and `.github/copilot-instructions.md` in the same change. If it changes a design fact, update ARCHITECTURE §5 in the `docs` repo too, and mention it to the user.

## Writing style

Plain, professional wording in docs and comments. No emoji, no marketing tone. Files are UTF-8 with LF line endings, following `.editorconfig`.
