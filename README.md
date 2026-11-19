# contracts

> **The Protobuf schema for the whole system**, published as a NuGet package for .NET and an npm package for TypeScript.

Part of [kubernetes-learning-platform](https://github.com/kubernetes-learning-platform/docs), a live Kubernetes lab with an optional AI helper. The [docs repository](https://github.com/kubernetes-learning-platform/docs) describes the overall system.

**Owner:** Filip · **Status:** the `v0.1` schema covers the whole lab as designed, so `api` and `web` build against real types even where the behaviour comes later. It follows the [v0.1 schema plan](https://github.com/kubernetes-learning-platform/docs/blob/main/plans/schema-v0.1.md).

---

## Overview

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="assets/contracts-dark.svg">
  <img src="assets/contracts-light.svg" width="100%" alt="The proto schema at the top publishes an npm package to web and NuGet packages to api, envmgr and the harness. Below, each boundary between those programs is labelled with the proto file that defines it: session and rest between web and api, envmgr between api and envmgr, harness between envmgr and the harness, and jobs through RabbitMQ.">
</picture>

The system consists of separate programs that exchange messages. This repository defines the structure of every one of those messages, once, in Protobuf.

The repository contains no runtime code. It produces two packages from the same schema:

- a **NuGet** package for the .NET programs (`api`, `envmgr` and the `harness`)
- an **npm** package for the TypeScript frontend (`web`)

Other repositories do not define message types by hand. They install the package and use the generated types, so an incompatible change is caught at compile time.

In the diagram, dashed lines are packages delivered to their consumers. Solid lines are the boundaries between programs, labelled with the defining `.proto` file and the transport. `topology` messages are carried inside other messages from the harness to the browser, and `view` messages are carried inside `session` messages from `api` to the browser.

### Change workflow

1. A field is added or changed in a `.proto` file through a pull request. Anyone can open it; Filip reviews it.
2. CI runs `buf lint` and `buf breaking`. A breaking change fails the PR unless its title marks it with `!`, as in `feat!: rename sandbox status`.
3. Merging publishes a release of both packages. The version comes from the PR title (see [Versioning](#versioning)).
4. A bot moves `sandbox`, `api` and `web` to the new version. In each one it builds and runs the tests, then pushes `build(deps): bump contracts to X.Y.Z` to `main`. Nobody writes a bump PR.
5. If a repo no longer builds, which only a breaking change can cause, the bot opens a PR there instead, and that repo's owner fixes the code in it.

## Why one schema

Every message that crosses a process boundary is defined here, once:

| Boundary | Transport | Messages |
|---|---|---|
| browser ↔ `api` | WebSocket, binary frames | `ClientMessage` / `ServerMessage` |
| browser → `api` | REST | request/response bodies |
| `api` ↔ `envmgr` | gRPC on the course VM, plain HTTP/2 | `EnvironmentManagerService` |
| `api` ↔ `envmgr` | RabbitMQ (sandbox start and teardown only) | `ProvisionRequest`, `ProvisionResult`, `TeardownRequest` |
| `envmgr` ↔ `harness` | WebSocket, one binary message per frame (every provider) | `HarnessFrame` |

A field change is one PR here, then a version bump in each consumer.

## Proto files

All under `proto/kubernetes_learning/v1/`:

| File | Defines |
|---|---|
| `common.proto` | `Error` and its `ErrorCode`; `Actor`, who caused something |
| `envmgr.proto` | The `EnvironmentManagerService` gRPC service, `Sandbox`, `SandboxStatus` |
| `harness.proto` | `HarnessFrame`: the hello (sandbox id + token), cluster status, exec, PTY, files, manifests and apply, faults, checks, and the topology stream |
| `topology.proto` | `Entity`, `Edge`, `TopologySnapshot`, `TopologyDiff`, `ClusterEvent`, `TrafficSample`, `Manifest` |
| `view.proto` | `ViewIntent`: focus, highlight, annotate, show_config, clear |
| `session.proto` | `ClientMessage` / `ServerMessage` for the browser socket, including `SessionStatus`, `HelperState`, helper questions (`AgentQuestion`, `AgentAnswer`) and `ScenarioProgress` |
| `rest.proto` | REST bodies: login, sessions, the scenario catalog and replay pages (`EventBatch`) |
| `jobs.proto` | `ProvisionRequest`, `ProvisionResult`, `TeardownRequest` for the RabbitMQ work queue |

Every file shares the package `kubernetes_learning.v1`, so a message name is unique across all of them. C# gets the namespace `KubernetesLearning.V1`; there are no file options.

### Conventions

- **IDs** are strings. Sessions, sandboxes, requests, questions and faults use UUIDs. A topology entity's ID is its Kubernetes UID, and an edge's is `<type>:<from_id>:<to_id>`.
- **Times** are `google.protobuf.Timestamp` and `google.protobuf.Duration`, which both packages map to native types.
- **Requests and replies:** a request carries a `request_id` chosen by the sender, and its reply carries it back, so replies match without ordering assumptions.
- **Failures** are an `Error`: a code, a message that is safe to show, and whether a retry can work. REST errors return it as the body.
- **REST bodies** use the proto3 JSON mapping. The replay pages (`EventBatch`) are binary `application/x-protobuf`.
- **Growth:** `ClientMessage`, `ServerMessage` and `HarnessFrame` grow through their `oneof`. Fields use proto3 `optional` where "not set" differs from zero.
- **TypeScript:** the `Error` message shadows the global `Error` when imported by name, so import it under another one: `import type { Error as ContractError } from "@kubernetes-learning-platform/contracts"`.

## Packages

| Package | Registry | Contents | Used by |
|---|---|---|---|
| `KubernetesLearning.Contracts` | GitHub Packages (NuGet) | Messages + gRPC client and server stubs, compiled by `Grpc.Tools` | `sandbox`, `api` |
| `@kubernetes-learning-platform/contracts` | GitHub Packages (npm) | Messages generated by protobuf-es | `web` |

The harness only uses the messages. Native AOT trimming drops the gRPC stubs.

## Versioning

Decided on 2026-11-14.

- Every merge to `main` that changes the schema or the package projects publishes a normal release of both packages. There are no prereleases and no hand-made tags.
- CI picks the version from the commit subject, which is the squashed PR title:

  | PR title | While on `0.x` |
  |---|---|
  | `feat!:` or `fix!:` (breaking) | Minor bump: `0.3.4` → `0.4.0` |
  | `feat:` or `fix:` | Patch bump: `0.3.4` → `0.3.5` |
  | Any other type (`docs:`, `ci:`, `chore:` …) | Nothing is published |

- CI pushes the `vX.Y.Z` tag itself. The first release is `v0.1.0`.
- `buf lint` and `buf breaking` (against `main`) gate every PR, and a break without `!` in the title fails it.
- After publishing, CI notifies `sandbox`, `api` and `web` with a GitHub App token. Each runs `platform`'s `contracts-bump.yml`, which sets the new version, builds, tests, and pushes to `main` with the same App's token, or opens a PR if the build fails. The App is on each repo's `main` ruleset bypass list, so the bump needs no PR; its key is an organization secret.
- The placeholder name has to be replaced **before 1.0**. After that, renaming the proto package is a breaking wire change. See [ARCHITECTURE §12](https://github.com/kubernetes-learning-platform/docs/blob/main/ARCHITECTURE.md#12-naming).

## Layout

```
contracts/
├── proto/kubernetes_learning/v1/      # the .proto files above
├── buf.yaml                           # lint + breaking rules
├── buf.gen.yaml                       # protobuf-es codegen
├── global.json                        # .NET SDK 10.0
├── dotnet/KubernetesLearning.Contracts/
│   └── KubernetesLearning.Contracts.csproj   # Grpc.Tools compiles ../../proto at build time
├── ts/
│   ├── package.json                   # @kubernetes-learning-platform/contracts
│   └── src/
│       ├── index.ts                   # re-exports each generated file
│       └── gen/                       # generated, not committed
└── .github/workflows/ci.yml           # calls platform's proto-ci.yml
```

## Installing the packages

GitHub Packages needs a token even to read. A consuming repo commits its registry config, which reads the token from `GITHUB_PACKAGES_TOKEN`, so nobody adds a package source by hand. [The contracts packages](https://github.com/kubernetes-learning-platform/docs#the-contracts-packages) in the docs repo shows how each person sets the variable.

**.NET:** `nuget.config` at the repo root, as in `sandbox` and `api`. The source mapping takes `KubernetesLearning.*` only from GitHub Packages and everything else from nuget.org:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" />
    <add key="github" value="https://nuget.pkg.github.com/kubernetes-learning-platform/index.json" />
  </packageSources>
  <packageSourceMapping>
    <packageSource key="nuget.org">
      <package pattern="*" />
    </packageSource>
    <packageSource key="github">
      <package pattern="KubernetesLearning.*" />
    </packageSource>
  </packageSourceMapping>
  <packageSourceCredentials>
    <github>
      <add key="Username" value="kubernetes-learning-platform" />
      <add key="ClearTextPassword" value="%GITHUB_PACKAGES_TOKEN%" />
    </github>
  </packageSourceCredentials>
</configuration>
```

**npm:** `.npmrc` at the repo root, as in `web`:

```ini
@kubernetes-learning-platform:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GITHUB_PACKAGES_TOKEN}
```

**CI:** the job grants `packages: read` and passes its `GITHUB_TOKEN` as `GITHUB_PACKAGES_TOKEN`, which `platform`'s `dotnet-ci` and `node-ci` already do. A new consuming repo also needs read access on both packages under **Manage Actions access**, or its restore fails with 403.

## Development

You'll need [buf](https://buf.build/docs/installation) (or `npx @bufbuild/buf`), the .NET 10 SDK and Node.js 24.

```bash
buf lint && buf format --diff --exit-code
(cd ts && npm ci) && buf generate
(cd dotnet/KubernetesLearning.Contracts && dotnet build -c Release)
(cd ts && npx biome ci . && npm run build)
```

A new proto file needs an `export *` line in `ts/src/index.ts`; the .NET project picks it up by itself.

On every PR, CI runs `platform`'s `proto-ci.yml`: the lint-copy check, buf lint, format and breaking checks, codegen, and both package builds.

## Further reading

- [ARCHITECTURE §5 Contracts](https://github.com/kubernetes-learning-platform/docs/blob/main/ARCHITECTURE.md#5-contracts)
- [ARCHITECTURE §6 Key flows](https://github.com/kubernetes-learning-platform/docs/blob/main/ARCHITECTURE.md#6-key-flows)
- [The v0.1 schema plan](https://github.com/kubernetes-learning-platform/docs/blob/main/plans/schema-v0.1.md), now history: the `.proto` files are the source of truth

## License

[MIT](LICENSE)
