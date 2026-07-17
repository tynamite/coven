# Readiness packet — Grok Build runtime PR (OpenCoven/coven-runtimes)

Status: **ready to open** — needs a contributor with OpenCoven push/PR access.
Prepared from the reviewed adapter branch `agent/grok-build-adapter` @ `7e73273`
(tynamite/coven PR #1, review cycle complete).

## What "the runtime PR" is

Two upstream changes, in order:

1. **OpenCoven/coven-runtimes** (this packet): extend the shared
   `coven-runtime-spec` with the `event_protocol` field and register the Grok
   Build manifest as an accepted, conformance-tested runtime. Tag as `v0.1.4`.
2. **OpenCoven/coven** (follow-up, separate PR): re-target the adapter branch
   from the fork, bump the `coven-runtime-spec` pin to `v0.1.4`, and move
   `event_protocol` deserialization onto the spec type.

Before opening either PR, follow the upstream claim protocol (`coven claim
status` + open-PR check, then `coven claim acquire`) — "someone has to make
this PR" is exactly the situation that has produced duplicate PRs before.

## 1. Spec delta (`coven-runtime-spec` v0.1.3 → v0.1.4)

Add one optional field to the adapter manifest spec:

- `event_protocol` (alias `eventProtocol`): optional enum selecting a reviewed
  machine-readable stdout translator for a **finite** headless process. First
  value: `grok-headless-v1`, matching Grok Build's public
  `--output-format streaming-json` emitter.
- Semantics: unlike `capabilities.stream` (long-lived bidirectional process),
  an event protocol describes a one-shot process per prompt; conversation
  continuity rides `continuity_args` cold-start resume.
- Validation rule: `event_protocol` and `capabilities.stream` are **mutually
  exclusive** (two process contracts cannot both own stdout). Reject manifests
  declaring both.

Reference implementation of the field, its serde shape, and the exclusion rule:
`crates/coven-cli/src/harness.rs` on the adapter branch (`HarnessEventProtocol`,
`ExternalHarnessAdapterSpec::event_protocol`, and the validation bail with its
negative test). Legacy manifests deserialize unchanged with the field absent.

## 2. Registry manifest — Grok Build

Content is authoritative from `GROK_BUILD_ADAPTER_MANIFEST` in
`crates/coven-cli/src/harness.rs` (unchanged across the review cycle). Confirm
file layout/naming against the registry's existing entries on checkout.

```json
{
  "adapters": [
    {
      "id": "grok",
      "label": "Grok Build",
      "executable": "grok",
      "interactive_prompt_prefix_args": ["--no-auto-update", "--no-alt-screen", "--output-format", "streaming-json"],
      "non_interactive_prompt_prefix_args": ["--no-auto-update", "--no-alt-screen", "--output-format", "streaming-json"],
      "install_hint": "Install Grok Build with `curl -fsSL https://x.ai/cli/install.sh | bash` or `npm install -g @xai-official/grok`; make sure `grok` is on PATH and run `grok login` (or set XAI_API_KEY for headless auth), then retry `coven adapter doctor grok`.",
      "system_prompt_flag": "--rules",
      "prompt_flag": "--single",
      "interactive_prompt_flag": "--single",
      "model_flag": "--model",
      "capabilities": {
        "stream": false,
        "preassigned_session_id": true,
        "think": false,
        "speed": false
      },
      "event_protocol": "grok-headless-v1",
      "sandbox": {
        "full_args": ["--permission-mode", "bypassPermissions", "--sandbox", "off"],
        "read_only_args": ["--permission-mode", "default", "--sandbox", "read-only"]
      },
      "continuity_args": {
        "init_prefix_args": ["--no-auto-update", "--no-alt-screen", "--output-format", "streaming-json"],
        "resume_prefix_args": ["--no-auto-update", "--no-alt-screen", "--output-format", "streaming-json"],
        "session_id_flag": "--session-id",
        "resume_flag": "--resume"
      }
    }
  ]
}
```

Upstream contract references for reviewers:

- Headless emitter source: https://github.com/xai-org/grok-build/blob/main/crates/codegen/xai-grok-pager/src/headless.rs
- CLI headless contract: https://docs.x.ai/build/cli/headless-scripting

## 3. Validation evidence (from the fork review cycle)

Automated, on the adapter branch:

- `cargo fmt --check`, `cargo clippy --workspace --all-targets -- -D warnings`,
  `cargo test --workspace --locked` (1,216 tests), secret guard, `git diff
  --check` — per PR #1 validation; fmt/secrets/whitespace re-verified
  independently at review time on each commit.
- Regression coverage added during review: mid-turn inactivity timeout (CLI +
  daemon bridges), SIGINT end-to-end supervision with descendant reaping and
  ledger state, session-id mismatch precedence, argv construction for fresh
  and resumed launches, auth-prompt fail-fast without device-code leakage,
  held-pipe bounding.

Live, against Grok Build 0.2.93 (macOS, 2026-07-16): authenticated read +
resume with matching session ids, read-only mutation rejection,
full-permission fixture write, invalid-model and invalid-key structured
failures without credential exposure, unauthenticated fail-fast without
device-code leakage, stale-session restore diagnostics, daemon replay,
process-tree cleanup. Details: `docs/harnesses/grok-build.md` §Current
maturity.

## 4. Known gaps to disclose in the upstream PR

- The CLI and daemon event-bridge runners share logic by duplication, not
  extraction (tracked in fork PR #1 review thread; extract before a second
  event protocol lands).
- Default permission for event-protocol adapters maps to the full policy
  (`bypassPermissions` + `--sandbox off`) when no `--permission` is given —
  including daemon/chat sessions, which currently have no read-only path.
  Deliberate, but must be documented in `docs/harnesses/grok-build.md` before
  or with the upstream PR.
- Windows auth fail-fast kills the direct PID only (Job Object +
  pipe-timeout backstop it).
- Grok's public headless schema has no tool-denied event; a denied action ends
  the turn without an explicit final denial (upstream limitation).
- Repeat live permission/failure checks on non-macOS platforms before any
  bundled-support claim (see maturity checklist in `docs/HARNESS-ADAPTERS.md`).

## 5. Draft PR body (coven-runtimes)

> ## Context
> - Summary: add `event_protocol` to the runtime manifest spec and register
>   Grok Build (`grok-headless-v1`) as an accepted runtime manifest.
> - Files changed: spec crate (+ conformance fixtures), Grok Build manifest.
> - Closes #<upstream issue — file one first>
>
> ## Implementation
> - Approach: optional, backward-compatible spec field; finite one-shot
>   process contract, mutually exclusive with `capabilities.stream`;
>   continuity via existing `continuity_args`. Reference implementation and
>   tests in the companion coven PR.
> - User-visible behavior: none in this repo; enables `coven adapter install
>   grok` consumers to validate against the canonical spec.
> - Compatibility notes: legacy manifests deserialize unchanged; new field is
>   `None` when absent. Tag `v0.1.4`.
>
> ## Verification
> - [ ] spec crate tests + conformance suite over the new manifest
> - [ ] mutual-exclusion negative case (`event_protocol` + `stream` rejected)
> - [ ] companion coven branch builds against the tagged spec
>
> ## Risk and Rollback
> - Risk level: low (additive optional field; opt-in runtime).
> - Rollback plan: revert tag; consumers stay pinned to v0.1.3.
>
> ## Agent Handoff
> - Current state: adapter implementation reviewed and hardened on the staging
>   fork (tynamite/coven#1: inactivity timeout, signal supervision, scoped
>   session-init hint, error precedence).
> - Follow-ups: companion OpenCoven/coven PR bumping the pin to v0.1.4.
> - Known gaps: see §4 of the packet.

## 6. Companion coven PR checklist

1. Claim the upstream issue; check for existing PRs.
2. Rebase `agent/grok-build-adapter` onto current `OpenCoven/coven` `main`.
3. Bump `coven-runtime-spec` pin to `v0.1.4`; replace the CLI-local
   `HarnessEventProtocol` with the spec type where they now overlap.
4. Add the permission-default documentation to `docs/harnesses/grok-build.md`
   (§4 above).
5. Run the full local gate set; fill the readiness template from this packet.
6. Preserve contributor attribution (numeric-id no-reply trailers) if the
   branch is squashed or re-landed.
