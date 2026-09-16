---
name: reasonblocks-setup
description: "Connect or migrate a Python application to ReasonBlocks dashboard capture, preserve task boundaries, and verify the local setup. Use when installing or troubleshooting the ReasonBlocks capture helper."
metadata:
  compatibility: Python 3.10 or newer, rbtrace 1.1.1, and an existing OpenAI or Anthropic Python SDK.
---

# ReasonBlocks setup

Connect the customer's existing Python application to dashboard capture. Inspect
its client construction and task boundaries, make the integration, and verify
what works locally.

Read the [setup guide](https://docs.reasonblocks.com/agent-setup.md) when installing
or migrating the client. Install the published `rbtrace==1.1.1` in the application's
environment and verify its commands there.

ReasonBlocks provides a CLI, a small Python integration helper and this skill.
The existing OpenAI or Anthropic Python client library—the provider SDK—continues
making model calls. Preserve the application's framework and client settings.

## Inspect and obtain the connection

Inspect the dependency manager, real client constructors, model-call paths,
logical task boundaries, launch command and deployment layout. A Python wrapper
may launch JavaScript or native code that actually makes the requests; a Python
helper does not instrument that child process. Integrate at the real boundary
and report unsupported paths.

Use the exact HTTPS source URL and key issued in **Data**. Current support is:

- OpenAI Chat Completions: `/capture/SOURCE_ID/openai/v1`, provider `openai`.
- Anthropic Messages: `/capture/SOURCE_ID/anthropic`, provider `anthropic`.

Do not invent a source URL, treat `rb_live_` organization/API keys as capture keys or
silently switch provider/API to fit this list. Responses, Gemini, Bedrock,
Fireworks and arbitrary upstreams are outside this dashboard setup.

Account sign-in and source/key creation currently happen in the dashboard.
Capture keys expire after seven days; rotation invalidates the previous key.
Connecting and ordinary capture require no sandbox or environment snapshot.

## Install and initialize

Use the project's package manager and record `rbtrace==1.1.1` in its dependency
manifest. Install it into the application environment; a global
or `uvx` tool environment alone is insufficient.

```bash
python -m rbtrace init --capture-url "$REASONBLOCKS_CAPTURE_URL" --provider anthropic --path . --json
```

Use `--provider openai` for an OpenAI source. `--dry-run` previews generated
files. The `rbtrace` entrypoint accepts the same arguments. Inspect existing
files before resolving conflicts; do not overwrite a customized helper.

Choose an importable helper location. `--path .` fits flat projects. An installed
`src/my_agent` package can use `--path src/my_agent` and
`from my_agent import reasonblocks_setup` or a package-relative import. A directly
launched script may see a helper beside it but not one at the repository root.
Do not add a `sys.path` workaround. Package the helper and its adjacent
`.reasonblocks/config.json` into the actual image or wheel package data; hidden
directories may otherwise be omitted. `.reasonblocks/SETUP.md` is optional at runtime.

Read the generated helper. Supply the capture key at runtime through
`REASONBLOCKS_CAPTURE_KEY`; the existing `RB_CAPTURE_KEY` alias is accepted, and
both must agree when set. Keep the provider credential in its existing secret
environment. Do not write secrets into CLI arguments, generated files or reports.

## Wire the application

Construct the existing `OpenAI`, `AsyncOpenAI`, `Anthropic` or `AsyncAnthropic`
client with `**reasonblocks_setup.client_kwargs()`. Merge other settings and
headers deliberately. These kwargs supply the exact source URL, capture auth
and `max_retries=0`. A timeout can leave paid work with an unknown outcome;
reconcile it before retrying. Preserve an intentional application retry policy
by explicitly overriding the returned kwargs, rather than adding a retry loop.

Importing the generated helper installs labeling on supported HTTP transports
and adds the configured capture host to the allowlist. `RBTRACE_HOSTS` entries
without a leading dot match exact hosts; `.example.com` matches subdomains.
The labels help group and order requests. They do not establish complete capture
or a replayable environment. `RBTRACE_DISABLE=1` disables labeling at startup;
`doctor` reports that condition.

Call `reasonblocks_setup.run_headers()` once at each logical task boundary and
pass the result as `extra_headers=headers` on every model call in that task.
Keep headers in each concurrent job's own state. Use `run_id=existing_task_id`
when a suitable task ID already exists. Never create a new ID per model step or
share one across unrelated tasks. Retain full messages, policy, tool definitions
and actual tool results; the application still executes tools. Existing explicit
`rbtrace.client.run()` task scopes may be retained instead. Without headers or
scopes, unscoped calls share one process-wide run.

Snapshots are optional for basic capture. If the application already has an
actual repeatable starting snapshot, use
`run_headers(snapshot_id=starting_snapshot_id)` and retain it throughout the task.
Do not invent a snapshot or label arbitrary state as resettable.

## Migrate an older generated connection

For a generated 1.1.0 connection, preserve its existing source URL and key.
For an older gateway setup, obtain a dashboard source URL and key first. Then:

```bash
python -m rbtrace migrate --capture-url "$REASONBLOCKS_CAPTURE_URL" --provider anthropic --path . --json
```

Use the old helper directory for `--path`, `openai` for an OpenAI Chat Completions
source, and `--dry-run` to preview. Migration backs up recognized generated files
and refuses customized helpers. Inspect and merge those customizations instead
of discarding them. It does not edit arbitrary application code.

Use current client construction and remove obsolete
`reasonblocks_setup.install()` calls; the generated helper installs labeling on
import. Keep explicit `rbtrace.client.run()` task scopes or pass fresh per-task
`run_headers()` on every call. Exact generated 1.1.0 files are recognized and
backed up before replacement.
Remove the old gateway base-URL setting from deployment configuration, then
restart/reconstruct affected clients and processes. A config-file rewrite alone
does not change an already-constructed client or its capture authentication.
Do not turn unsupported providers or Responses calls into Anthropic/Chat
Completions as an incidental migration. Report the compatibility gap.

## Verify and report

Run `python -m rbtrace doctor --path . --json` in the actual application
environment, using the same helper path as initialization. For uv, use
`uv run python -m rbtrace doctor ...`. It checks local configuration, SDK
availability, capture-key presence and selected SDK transport labeling. It
makes no network calls and does not verify provider credentials, remote key
validity or delivered captures.

Inspect the application diff, run relevant existing local tests, and test the
actual launch from outside the repository directory to verify imports and
config packaging. Live model calls send application content and incur provider
charges; run one only within the user's authorized scope. Confirm source records
in **Data** before claiming end-to-end success.

Report changed files, selected source/provider, task-boundary placement, local
checks and any missing inputs. A request to install is not itself authorization
to spend on training or change production traffic.

Capture requires no sandbox. Training later needs representative tasks and a
resettable test environment with an independent outcome evaluator. The helper
does not create snapshots or register that environment. Link to
[full-agent training](https://docs.reasonblocks.com/full-agent-training.md) when
those prerequisites are relevant.
