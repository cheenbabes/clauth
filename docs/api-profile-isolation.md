# API profile isolation

An API profile and a subscription profile use different authentication sources.
A saved conversation can resume under either without changing its transcript ID.
`clauth resume <session-id> --profile <api-profile>` uses a per-session runtime;
it does not select the API profile as the global active account.

## The resume path

1. `sessions_cli::run_resume` resolves the transcript and its recorded workspace,
   resolves the requested profile, and passes `--resume <id>` into `start::run`.
   Its `follows_chain` argument is false.
2. `start::run` checks admission, acquires a `ProfileRuntime`, scrubs the managed
   authentication environment, and sets `CLAUDE_CONFIG_DIR` to that runtime.
3. `runtime::write_merged_settings` builds a real runtime settings file from the
   shared settings and the destination profile. It does not replace the global
   credential slot. `claude::build_runtime_claude_settings_json` drops inherited
   `ANTHROPIC_AUTH_TOKEN` and `ANTHROPIC_API_KEY`, then applies explicit env
   entries belonging to the destination profile. A profile's `api_key` is
   exposed through `clauth __api-key <profile>` as `apiKeyHelper`.
4. `runtime::seed_claude_json` copies shared configuration into a fresh runtime,
   excluding `oauthAccount`, `primaryApiKey`, and `customApiKeyResponses`.
   The source file is unchanged. Existing runtime copies retain their own
   authentication fields.
5. Shared-mode runtimes link the conversation store, commands, and plugins.
   `settings_sync` and `claude_json` synchronize shared preferences while
   retaining authentication fields on their owning member. The API helper and
   login-managed API key are not synchronized into subscription runtimes.
6. The runtime guard owns shutdown and transcript preservation. None of this
   selects a different `profiles.toml` global active profile or joins the API
   profile to the subscription fallback chain.

## Global switching is a different operation

`actions::switch_profile_cli` settles the outgoing macOS OAuth Keychain login,
checks that the destination's stored credentials are installable, and invokes
`switch_profile_gated`. The locked switch snapshots outgoing credentials,
installs the destination's credential slot, writes global settings, and changes
`active_profile`. A credential-less API installation signs out the global
OAuth slot; switching back reinstalls the subscription's saved login. The
credential synchronizer retires the old owner when API authentication is
installed, preventing its queued retry from undoing the switch.

This path does not fully manage a separate API credential saved by Claude's
`/login`. In particular, the post-switch identity cleanup removes `oauthAccount`
but leaves `primaryApiKey`. On macOS, Claude also has a separate API-key
Keychain item, distinct from its OAuth credential item. Removing only the JSON
field is neither a complete migration nor a safe way to preserve an unmanaged
credential. The global settings builder also retains an unmanaged
`ANTHROPIC_API_KEY` that is not owned by the outgoing profile. This patch does
not delete or migrate those credentials and does not claim to repair all
global API-to-subscription transitions.

## Scope and limits

The fixes cover two reproducible leaks into runtime copies: a key inherited
from the shared settings base, and a saved login key copied into a fresh runtime. Explicit
API-key env entries on the selected profile still work. Existing runtime files
and the global login-managed key are not rewritten by the seed fix.

A parent shell, project settings, managed policy, or an explicit command-line
settings override can still select another authentication source. The managed
environment scrub is not an allowlist of every Claude authentication variable;
for example, a caller-supplied `CLAUDE_CODE_OAUTH_TOKEN` or
`CLAUDE_SECURESTORAGE_CONFIG_DIR` needs separate consideration. Inspect the
session's actual authentication source before spending.

Tests use synthetic values and disposable home directories. The ordinary test
build disables the real macOS Keychain adapter; passing these tests does not
establish a live Keychain round trip or permission to alter a user's accounts.
