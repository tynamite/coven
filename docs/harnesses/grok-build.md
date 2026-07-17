---
summary: "Experimental Grok Build adapter recipe for running xAI's coding-agent CLI through Coven."
read_when:
  - Installing the Grok Build adapter
  - Reviewing Grok Build launch, permission, and session behavior
title: "Grok Build (experimental)"
description: "Install and use Coven's trusted Grok Build adapter recipe without promoting Grok to a bundled default harness."
---

Grok Build is available through a trusted, installable Coven adapter recipe. It is **not** a bundled default harness yet: users opt in with `coven adapter install grok`, and the recipe stays experimental until its remaining permission, failure-path, and client-compatibility checks pass the [adapter maturity checklist](/HARNESS-ADAPTERS#suggested-adapter-maturity-stages).

The integration is based on xAI's Apache-2.0 [Grok Build source](https://github.com/xai-org/grok-build), including its headless emitter and session-startup rules. Coven does not embed or fork Grok Build; it launches the installed CLI and translates its public protocol.

The Coven harness id is `grok`; the executable is `grok`.

## Install

<Steps>
  <Step title="Install and authenticate Grok Build">
    Use either installation path documented by xAI:

    ```bash
    curl -fsSL https://x.ai/cli/install.sh | bash
    # or
    npm install -g @xai-official/grok
    ```

    Then authenticate with the CLI:

    ```bash
    grok login
    # Headless or remote machine:
    grok login --device-auth
    ```

    Grok also supports `XAI_API_KEY` for headless automation. Coven does not read, store, or inject that credential; Grok resolves it from its own inherited environment.
  </Step>
  <Step title="Install the trusted Coven recipe">
    ```bash
    coven adapter install grok
    coven adapter doctor grok
    ```

    The first command writes the versioned recipe to `COVEN_HOME/adapters/grok.json`. Coven only loads the file while it exactly matches its bundled trusted recipe. `adapter doctor` also runs Grok's local help command and verifies that the installed binary advertises every flag required by the adapter.
  </Step>
  <Step title="Run a project-scoped session">
    ```bash
    cd /path/to/project
    coven run grok "explain this repository"
    ```
  </Step>
</Steps>

## Adapter contract

| Coven behavior | Grok Build argv |
|---|---|
| One-shot prompt | `--single=<prompt>` |
| Model selection | `--model <model>` |
| Familiar identity | `--rules <identity>` |
| New named conversation | `--session-id <uuid>` |
| Resume conversation | `--resume <uuid>` |
| Machine-friendly output | `--no-alt-screen --output-format streaming-json` |
| Deterministic startup | `--no-auto-update` |

The prompt is bound with the long flag's `=` form and remains the final argv entry. A prompt beginning with `-` therefore stays user data and cannot become a Grok CLI option. Coven launches the executable directly and never constructs a shell command string.

Grok Build's source defines a newline-delimited headless protocol. Coven runs that finite process on ordinary pipes and translates it as follows:

| Grok event | Coven behavior |
|---|---|
| `text` | forwards `data` as assistant output |
| `thought` | validates and discards it; private reasoning is never copied into Coven output or replay logs |
| `end` | captures `sessionId`, `requestId`, and `stopReason` as terminal transport metadata, then verifies that `sessionId` matches the UUID Coven assigned |
| `error` | fails the Coven session with the native message |
| unknown event | safely ignored for forward compatibility |

This is not Claude-style long-lived stream mode. Every prompt starts one finite `grok --single` process. The first turn receives `--session-id <Coven UUID>`; a later turn cold-starts with `--resume <same UUID>`. The same translation runs for direct `coven run`, `--stream-json`, and daemon/chat sessions.

The terminal `requestId` and `stopReason` are validated by the bridge but are not emitted as assistant text or copied into Coven replay events.

## Permissions

`coven run --permission` maps to both Grok's permission policy and its process sandbox:

| Coven policy | Grok Build mapping |
|---|---|
| `full` | `--permission-mode bypassPermissions --sandbox off` |
| `read-only` | `--permission-mode default --sandbox read-only` |

When `--permission` is omitted, Coven's policy defaults to `full`. For Grok Build, Coven applies that default explicitly by launching `--permission-mode bypassPermissions --sandbox off`; it does not leave the decision to Grok's native defaults. This allows the headless turn to proceed without Grok permission prompts and disables Grok's process sandbox, so treat an omitted permission flag as an explicit full-trust choice.

Daemon and chat launches currently use this full mapping on every turn. The daemon session API does not expose a permission field, so daemon and chat clients cannot request Grok's read-only mapping yet. For review and audit work that requires read-only behavior, use a direct foreground launch:

```bash
coven run grok "review this repository" --permission read-only
```

Grok's public source accepts `plan` as a compatibility value on the command line but does not activate a plan permission policy from that value, so the adapter explicitly selects `default` and relies on the native read-only sandbox for the filesystem boundary. Grok's own documentation notes that child-process network blocking in restrictive sandbox profiles is currently enforced on Linux but not macOS; treat that platform limitation as part of Grok's boundary, not a guarantee supplied by Coven.

Grok Build does not document a native additional-directory flag, so `coven run grok --add-dir ...` is a warned no-op. Start the session at the intended project root instead.

## Current maturity

This recipe covers:

- safe command construction for normal and `-`-prefixed prompts;
- `coven adapter install` and a headless-contract probe in `coven adapter doctor`;
- model forwarding;
- full and read-only permission mapping;
- source-schema JSONL parsing with thought suppression and terminal-id validation;
- preassigned session ids and resume argv;
- direct CLI, Coven `--stream-json`, and daemon event-replay smoke tests against a fake runtime that emits the published schema;
- foreground and daemon fail-fast guards when Grok attempts interactive device authentication during a headless run; and
- bounded post-exit pipe draining with process-tree cleanup when a harness descendant keeps stdout or stderr open.

Manual provider smoke tests passed on 2026-07-16 with Grok Build 0.2.93 on macOS:

- a read-only first turn read a disposable fixture through `coven run --stream-json`, and a cold-start `--continue` turn recalled the prior response without rereading the file;
- Grok returned the UUID Coven assigned on both turns;
- a read-only mutation attempt in a non-temporary nested repository left the requested file absent;
- a full-permission plain-output turn created exactly the requested 16-byte fixture and returned the expected response;
- a deliberately invalid model produced a normalized Coven error result and exit code 1 without assistant output;
- an isolated invalid API key produced a structured provider error and exit code 1 without exposing the supplied key;
- continuing a session deliberately never created in Grok preserved its remote restore failure and 404 in Coven's structured error result;
- a real daemon/API launch forced an interactive request through the finite headless bridge, reconstructed the expected response in replay events, suppressed native thought/end frames, and persisted exit code 0; and
- isolated unauthenticated foreground and daemon launches both failed in about two seconds with a normalized exit code 1 and remediation message, without exposing Grok's device URL or code.

One permission-denial UX caveat remains: the denied live turn stopped after preliminary assistant text instead of emitting an explicit final denial, although the filesystem boundary held. Grok's public headless schema does not expose a dedicated tool-denied event for Coven to translate.

Before describing Grok Build as bundled support, maintainers should repeat the permission and failure checks on other supported operating systems, exercise expired-credential and rate-limit failures, and validate supported client flows. Real-provider tests can transmit project content and consume account quota, so they remain separate from contributor-safe CI.

## Upstream references

- [Grok Build getting started](https://docs.x.ai/build/overview)
- [Grok Build source](https://github.com/xai-org/grok-build)
- [Headless emitter source](https://github.com/xai-org/grok-build/blob/main/crates/codegen/xai-grok-pager/src/headless.rs)
- [Headless and scripting](https://docs.x.ai/build/cli/headless-scripting)
- [CLI reference](https://docs.x.ai/build/cli/reference)
- [Sandbox and permission controls](https://docs.x.ai/build/enterprise)

## Related

- [Harnesses](/harnesses)
- [Harness adapter guide](/HARNESS-ADAPTERS)
- [Provider auth boundary](/harnesses/provider-auth)
