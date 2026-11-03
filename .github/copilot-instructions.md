# Copilot instructions: contracts

This repo holds the Protobuf schema for kubernetes-learning-platform, an AI tutor with a live Kubernetes lab. Every message that crosses a process boundary is defined here: browser ↔ api (WebSocket), api ↔ envmgr (gRPC), envmgr ↔ harness (vsock frames), and REST bodies. The full design is in the `docs` repo (ARCHITECTURE.md §5).

## Layout
- `proto/kubernetes_learning/v1/`: `common`, `envmgr`, `harness`, `topology`, `view`, `session`, `rest`.
- `dotnet/KubernetesLearning.Contracts/`: NuGet package. `Grpc.Tools` compiles the protos at build time.
- `ts/`: npm package `@kubernetes-learning-platform/contracts`, generated with protobuf-es. `ts/src/gen/` is generated and never committed.

## When reviewing, focus on
- **Wire compatibility.** Flag changed field numbers or types, reused numbers, and removed fields or enum values without a `reserved` entry for both the number and the name. While on `0.x`, a breaking change is allowed but needs a minor version bump. Say so explicitly.
- **buf lint style.** Messages and enums in PascalCase, fields in snake_case, the first enum value is `<ENUM>_UNSPECIFIED = 0`, and enum values are prefixed with the enum name.
- **Placement.** A new message goes in the file for its boundary (e.g. browser socket messages in `session.proto`). `ServerMessage` / `ClientMessage` / `HarnessFrame` grow through their `oneof`.
- **Comments.** Every message and every non-obvious field has a short comment. Consumers read these as the API docs.
- **Generated code** committed by mistake (`ts/src/gen/`, `bin/`, `obj/`).
- **Naming.** The proto package must stay `kubernetes_learning.v1`. The name is a placeholder that gets renamed before 1.0, so don't hardcode it anywhere new.

## Project-wide rules
- PR titles are Conventional Commits (`feat: short description`), one line, at most 72 characters, no trailing period. The title becomes the squash commit on `main`.
- Files are UTF-8 with LF line endings (`.editorconfig`, `.gitattributes`).

## Don't
- Don't comment on formatting that `buf format` or `.editorconfig` already handles.
- Keep reviews short. Skip praise and summaries of the diff.
