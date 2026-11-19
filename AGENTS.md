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
- **Status:** the v0.1 schema is in `proto/` (contracts#8), from the `docs` repo's `plans/schema-v0.1.md`. Every schema merge to `main` publishes both packages (contracts#2).
- **Design:** `README.md` here, and ARCHITECTURE §5 in the `docs` repo. Where they disagree, the `docs` repo wins.

## Layout

```
contracts/
├── proto/kubernetes_learning/v1/          # common, envmgr, harness, topology, view, session, rest, jobs
├── buf.yaml                               # lint (STANDARD) + breaking (FILE) rules
├── buf.gen.yaml                           # protobuf-es codegen into ts/src/gen/
├── global.json                            # SDK 10.0
├── dotnet/KubernetesLearning.Contracts/   # NuGet; Grpc.Tools compiles ../../proto at build time
├── ts/                                    # npm package; src/index.ts re-exports src/gen/, which is never committed
└── .github/workflows/                     # ci.yml calls platform's proto-ci.yml
```

Scripts, when there are any, go in `scripts/`.

## Build and check

You need the buf CLI (CI uses 1.75.0; `npx @bufbuild/buf@1.75.0` works without installing it), the .NET 10 SDK and Node.js 24. Before saying a change is done, run what CI runs, from the repo root unless noted:

- `buf lint` and `buf format --diff --exit-code`
- `buf breaking --against '.git#branch=main'`
- `npm ci` in `ts/`, then `buf generate` (it uses `ts/node_modules/.bin/protoc-gen-es`)
- in `dotnet/KubernetesLearning.Contracts/`: `dotnet format --verify-no-changes` and `dotnet build -c Release`
- in `ts/`: `npx biome ci .`, `npm run build` and `npx tsc --noEmit`
- `../platform/lint/sync.sh --check --dotnet --node .`, with `platform` at the commit in `.lint-sync`

CI (`.github/workflows/ci.yml`) reports `ci / Protobuf checks`, required in the `main` ruleset. It also runs on title edits, because the title decides whether a break is allowed. `Grpc.Tools` 2.85.0 ships a universal macOS `protoc`, so the .NET build works on Apple silicon; don't go below it.

**New proto files.** Add an `export * from "./gen/kubernetes_learning/v1/<file>_pb.js";` line to `ts/src/index.ts` for each one. The .NET project picks up every file under `proto/` by itself.

**Lint configs.** `.editorconfig`, `Directory.Build.props`, `biome.json` and `.lint-sync` are copies from `platform/lint/`. Don't edit them here; change `platform/lint/` instead, and CI fails a hand edit. They were made with `lint/sync.sh --dotnet --node`. Keep `ts/src/gen/` in `.gitignore`: that is also how Biome skips the generated code. Pin `@biomejs/biome` to the exact version in `biome.json`'s `$schema` (2.6.4). New lint configs arrive as bot commits, `chore: sync lint configs`, which never publish; if the new rules break the build, the bot opens a PR instead.

## Schema rules

- **Wire compatibility.** Never change a field's number or type, or reuse a number. Removing a field or enum value needs a `reserved` entry for both the number and the name.
- **Style.** Messages and enums in PascalCase, fields in snake_case. The first enum value is `<ENUM>_UNSPECIFIED = 0`, and every value is prefixed with the enum name.
- **Placement.** A message goes in the file for its boundary. `ClientMessage`, `ServerMessage` and `HarnessFrame` grow through their `oneof`. Every file shares one package, so a message or enum name must be unique across all of them.
- **Conventions.** IDs are strings (UUIDs, or Kubernetes UIDs for topology entities). Times are `google.protobuf.Timestamp` and `Duration`. A request carries a sender-chosen `request_id` and its reply carries it back. Failures use `Error` from `common.proto`. REST bodies use the proto3 JSON mapping. `README.md` lists the details.
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
- **Where changes land:** every change goes through a PR from the issue's branch. README, `AGENTS.md` and other doc updates go in the PR of the feature they describe, not in separate commits. When a change in another repo makes this repo's docs wrong, fix them in a small `docs:` PR here. Only the `docs` repo commits straight to `main`.
- **PRs:** the title follows the commit format, at most 72 characters; it becomes the squash commit on `main`. The description is one or two plain sentences, plus `Closes #<number>`. Merging needs one approval from someone other than the author, and the branch must be up to date with `main`. Filip is requested as reviewer on every PR (`.github/CODEOWNERS`), and the author is assigned automatically. Only Filip merges.
- **Public repo:** everything committed is public at once. Never commit secrets, real `.env` files or personal data; push protection only catches known token formats.

## Keep docs in sync

When a change touches this repo's design, layout, commands or conventions, update `AGENTS.md`, `README.md` and `.github/copilot-instructions.md` in the same change. If it changes a design fact, update ARCHITECTURE §5 in the `docs` repo too, and mention it to the user.

## Writing style

Plain, professional wording in docs and comments. No emoji, no marketing tone. Files are UTF-8 with LF line endings, following `.editorconfig`.
