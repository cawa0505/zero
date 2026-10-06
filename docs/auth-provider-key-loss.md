# Auth Provider Key Loss: Investigation Handoff

## Status

Upstream tracking. Upstream branch `fix-auth-propagation-gaps` (commit `10ef5f55`)
systematically addresses this issue by consolidating credential resolution across
all provider-build surfaces (sub-agents, ACP sessions, escalation, etc.) and
wrapping `deps.newProvider` once in `fillAppDeps`.

Our local fork maintains this analysis for reference, while tracking upstream
fixes as they land in mainline releases.

## Current Candidate Change

`internal/tui/session.go` currently calls `config.ApplyStoredAPIKey()` from
`ensureActiveSession()` after creating new session metadata:

```go
store, err := m.providerKeyStore()
if err == nil {
	m.providerProfile = config.ApplyStoredAPIKey(m.providerProfile, store)
}
```

This change builds successfully, but it is not proven to fix the reported
behavior.

Important risk: the call only mutates `m.providerProfile`. If `m.provider` was
already constructed before `ensureActiveSession()`, reloading the key at this
point may not update the provider used for the request. The actual prompt-to-
provider construction sequence must be traced before keeping this patch.

## Confirmed Code Facts

- `config.SecureProviderProfile()` stores an inline key and clears it from the
  profile only after the store write succeeds.
- `config.ApplyStoredAPIKey()` reads a key only when `APIKeyStored` is true and
  `APIKey` is empty. Inline or environment-resolved keys take precedence.
- `model.providerKeyStore()` opens the store beside `m.userConfigPath`; an empty
  path uses the default user credential store.
- `startNewSession()` clears conversation/session state but intentionally
  preserves model and provider configuration.
- Stored-key resolution already exists in several paths, including
  `profileWithCredential()`, provider/model pickers, provider discovery, and CLI
  execution.
- The credential store delegates `Get`, `Set`, and `Delete` to one selected
  backend. It is not an in-memory cache that must be repopulated on every TUI
  session.

These facts invalidate the earlier theory that `credstore.Get()` needs a generic
keyring fallback. The selected backend already owns the read operation.

## Evidence Collected

- `go build ./...`: passed after the candidate change.
- `go test ./internal/config/... -run TestApplyStoredAPIKey`: passed.
- Provider-focused TUI tests passed.
- Full `go test ./internal/tui/...` had an unrelated existing failure in
  `TestAltScreenTranscriptScrollKeepsFooterFixed`.
- A Zero `0.7.0` binary containing the candidate change was built with
  `go run ./cmd/zero-release build` and installed at
  `~/.local/bin/zero`.
- The installed binary hash at installation time was
  `cb9e72ecac2182453a58ff27018284456cafb0a3c3f06c42404f5741f8ec2cd4`.

The earlier `go build -o zero ./cmd/zero-release` command built the release tool,
not the Zero application. That artifact was replaced by the proper release build.

## Required Reproduction

Use a disposable provider profile and non-production key:

1. Record the active user config path, provider profile, `APIKeyStored` value,
   selected credential backend, and credential-store directory.
2. Save the key through the same TUI flow reported by the user.
3. Confirm `store.Get(profile.Name)` succeeds before `/new` without printing the
   secret.
4. Run one authenticated request in the current session.
5. Execute `/new` and run the same request again.
6. Confirm whether `store.Get(profile.Name)` still succeeds after `/new`.
7. If the read succeeds but the request fails, trace provider construction and
   replacement; the problem is runtime wiring, not persistence.
8. If the read fails, compare store backend, directory, and provider account name
   between write and read paths.
9. Repeat with a full process restart only after the `/new` boundary is understood;
   these are different lifecycle boundaries.

Do not log or persist the key value. Record only backend, path, provider/account
name, and present/missing state.

## Next Code Investigation

1. Trace callers around `ensureActiveSession()`, `profileWithCredential()`, and
   provider construction to establish when `m.provider` receives credentials.
2. Add one regression test that saves a fake key, starts a new TUI session, and
   proves the provider factory receives that key on the next prompt.
3. Keep the candidate patch only if that test fails without it and passes with it.
4. Otherwise revert the patch and fix the first confirmed divergence between the
   key write path and the provider construction path.

## Working Tree Scope

Relevant uncommitted files at handoff:

- `internal/tui/session.go`: candidate implementation, not yet proven.
- `docs/auth-provider-key-loss.md`: this handoff.

No commit was created.
